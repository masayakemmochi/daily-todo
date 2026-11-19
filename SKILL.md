---
name: daily-todo
description: 朝の業務開始時に Slack（自分宛メンション・DM）と Jira（担当・期限・ウォッチ）を集約し、今日やるべきことをサマリして自分宛 Slack DM に送る。「今日やること」「朝のサマリ」「デイリーTODO」「/daily-todo」で起動する。
---

# daily-todo

Slack と Jira から「今日自分がやるべきこと」を抽出し、自分宛 Slack DM に送る。

## 0. 設定読み込み

1. `~/.claude/skills/daily-todo/config.md` を読む。存在しなければ同じディレクトリの `config.example.md` をコピーして作る。個人の ID をリポジトリに含めないため、`config.md` は git 管理外にしている。
2. `SLACK_USER_ID` または `SLACK_SELF_DM_ID` が空なら初回セットアップを行う。
   - `mcp__claude_ai_Slack__slack_search_users` の説明文に記載されているログインユーザーの ID を `SLACK_USER_ID` とする。
   - `mcp__claude_ai_Slack__slack_list_user_channels`（`types: im`）で自分自身との DM を探し、その ID を `SLACK_SELF_DM_ID` とする。見つからなければ `SLACK_USER_ID` を入れる。`slack_send_message` はユーザー ID を指定すると、そのユーザーとの DM に送るため。
   - 取得値を `config.md` に書き込む。
3. 収集期間の起点を決める。月曜は前週金曜 `WORKDAY_END`、それ以外は前日 `WORKDAY_END`。

## 1. Slack 収集

ツールはすべて claude.ai の Slack 連携 `mcp__claude_ai_Slack__*`（ToolSearch で読み込む）。自作 Slack アプリ用の `mcp__slack__*` は廃止したため使わない。

時刻の引数には起点の Unix タイムスタンプ（以下「起点 ts」）を渡す。

- **メンション**：`slack_search_public_and_private` で `keywords: ["<@SLACK_USER_ID>"]`、`after: 起点 ts`、`sort: timestamp` を指定して検索する。1 回 20 件までのため、`cursor` が返る間はページを送る。
- **DM / グループ DM**：`slack_list_user_channels`（`types: im,mpim`、`exclude_archived: true`）で一覧を取り、各チャンネルを `slack_read_channel`（`oldest: 起点 ts`）で読む。自分の DM（`SLACK_SELF_DM_ID`）は除外する。過去の daily-todo 投稿を拾わないため。
- 依頼・質問を含むメッセージは**自分が既に返信済みか**を判定し、返信済みのものは除外する。
  - スレッド内のメッセージは `slack_read_thread` で返信を読み、メンションより後の時刻に自分の投稿があれば返信済みとみなす。
  - スレッド外のメッセージは、同じチャンネルでメンションより後の時刻に自分の投稿があれば返信済みとみなす。DM は `slack_read_channel` の結果で判定する。

Slack 連携が未接続、または権限エラーになった場合は処理を止めず、サマリ末尾の「取得失敗」に記載する。Slack が使えないときも Jira の分だけは届けるため。

## 2. Jira 収集

`mcp__atlassian__jira_search` を使う。

| 区分 | JQL |
|---|---|
| 担当・未完了 | `assignee = currentUser() AND statusCategory != Done ORDER BY priority DESC, duedate ASC` |
| 期限接近 | `assignee = currentUser() AND statusCategory != Done AND duedate <= {DUE_SOON_DAYS}d ORDER BY duedate ASC` |
| ウォッチ更新 | `watcher = currentUser() AND updated >= -{起点からの日数}d ORDER BY updated DESC` |

ウォッチ更新の課題は、起点以降のコメントに自分へのメンションがあるかを確認し、メンションがあれば「返信・確認待ち」に入れる。

## 3. 分類と優先度付け

| 区分 | 入れる条件 |
|---|---|
| 🔴 今日やる | 期限が今日以前の Jira／メンションで今日中の対応を求められているもの／Priority が Highest・High の担当課題 |
| 🟡 返信・確認待ち | 未返信のメンション・DM・Jira コメントでのメンション |
| 🔵 期限が近い | `DUE_SOON_DAYS` 以内に期限が来る担当課題（🔴 に入れたものは除く） |
| ⚪ 参考 | 上記以外の担当未完了課題（全件を列挙する） |

同じ件が Slack と Jira の両方に出た場合は 1 項目にまとめる。

## 4. サマリ書式（標準 Markdown）

`slack_send_message` は標準 Markdown を Slack の書式に変換する。Slack 独自の mrkdwn（`*太字*`、`<URL|text>`）を書くと変換されずに崩れるため、標準 Markdown で書く。

```
**📋 {YYYY-MM-DD(曜)} 今日やること**

**🔴 今日やる（N件）**
- [KEY-123](URL) 要約（期限: MM/DD）
- [#channel @相手](permalink) 依頼内容の要約

**🟡 返信・確認待ち（N件）**
- ...

**🔵 期限が近い（N件）**
- ...

**⚪ 参考（N件）**
- [KEY-123](URL) 要約（優先度: Medium／期限: MM/DD または なし）

_取得失敗: Slack メンション検索（権限不足）_   ← 失敗時のみ
```

- 1 つのテキスト要素は 5000 字までのため、超える場合は ⚪ 参考を件数のみに省略する。

- 1 項目 1 行。要約は 40 字以内で、何をすべきかを動詞で書く。
- 各項目に元のリンクを必ず付ける。
- 該当 0 件の区分は「なし」と書く。

## 5. 送信と表示

1. `mcp__claude_ai_Slack__slack_send_message` で `channel_id = SLACK_SELF_DM_ID` に投稿する。確認は挟まない。
   - **送信先は `SLACK_SELF_DM_ID` 以外にしてはならない。** 他者に誤送信しないため。
   - ツールが存在しない場合は送信をスキップし、その旨を端末に表示する。
2. 同じ内容を端末にも表示する。

## 定時実行の設定

ユーザーから「毎朝自動で」と言われたら、`CronCreate` で以下を登録する（セッション起動中のみ有効）。

- cron: `0 9 * * 1-5`（平日 9:00）
- prompt: `/daily-todo`
