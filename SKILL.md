---
name: daily-todo
description: 朝の業務開始時に Slack（自分宛メンション・DM）と Jira（担当・期限・ウォッチ）を集約し、今日やるべきことをサマリして自分宛 Slack DM に送る。「今日やること」「朝のサマリ」「デイリーTODO」「/daily-todo」で起動する。
---

# daily-todo

Slack と Jira から「今日自分がやるべきこと」を抽出し、自分宛 Slack DM に送る。

## 0. 設定読み込み

1. `~/.claude/skills/daily-todo/config.md` を読む。存在しなければ同じディレクトリの `config.example.md` をコピーして作る。個人の ID をリポジトリに含めないため、`config.md` は git 管理外にしている。
2. `SLACK_USER_ID` または `SLACK_SELF_DM_ID` が空なら初回セットアップを行う。
   - `mcp__slack__users_search` で自分を検索し（ユーザーに表示名を聞く）、ユーザー ID と DM チャンネル ID を取得する。
   - 取得値を `config.md` に書き込む。
3. 収集期間の起点を決める。月曜は前週金曜 `WORKDAY_END`、それ以外は前日 `WORKDAY_END`。

## 1. Slack 収集

ツールはすべて `mcp__slack__*`（ToolSearch で読み込む）。

- **メンション**：`conversations_search_messages` で `<@SLACK_USER_ID>` を起点日以降で検索する。
- **DM / グループ DM**：`channels_me`（`channel_types: im,mpim`）で一覧を取り、各チャンネルを `conversations_history`（`limit` は起点からの日数）で読む。自分の DM（`SLACK_SELF_DM_ID`）は除外する。過去の daily-todo 投稿を拾わないため。
- 依頼・質問を含むメッセージは**自分が既に返信済みか**を判定し、返信済みのものは除外する。
  - `conversations_search_messages` で `filter_users_from: SLACK_USER_ID`、起点日以降の自分の投稿を検索する。
  - 同じ `ThreadTs` を持ち、メンションより後の時刻の自分の投稿があれば返信済みとみなす。
  - `conversations_replies` は使わない。現在のトークンでは missing_scope になるため。この判定方法は通常手順なので「取得失敗」には書かない。

検索ツールが存在しない、または権限エラーになった場合は処理を止めず、サマリ末尾の「取得失敗」に記載する。xoxp 未設定時に Jira だけでも届けるため。

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

## 4. サマリ書式（Slack mrkdwn）

```
*📋 {YYYY-MM-DD(曜)} 今日やること*

*🔴 今日やる（N件）*
• <URL|KEY-123> 要約（期限: MM/DD）
• <permalink|#channel @相手> 依頼内容の要約

*🟡 返信・確認待ち（N件）*
• ...

*🔵 期限が近い（N件）*
• ...

*⚪ 参考（N件）*
• <URL|KEY-123> 要約（優先度: Medium／期限: MM/DD または なし）

_取得失敗: Slack メンション検索（権限不足）_   ← 失敗時のみ
```

- 1 項目 1 行。要約は 40 字以内で、何をすべきかを動詞で書く。
- 各項目に元のリンクを必ず付ける。
- 該当 0 件の区分は「なし」と書く。

## 5. 送信と表示

1. `mcp__slack__conversations_add_message` で `channel_id = SLACK_SELF_DM_ID` に投稿する。確認は挟まない。
   - **送信先は `SLACK_SELF_DM_ID` 以外にしてはならない。** 他者に誤送信しないため。
   - ツールが存在しない場合は送信をスキップし、その旨を端末に表示する。
2. 同じ内容を端末にも表示する。

## 定時実行の設定

ユーザーから「毎朝自動で」と言われたら、`CronCreate` で以下を登録する（セッション起動中のみ有効）。

- cron: `0 9 * * 1-5`（平日 9:00）
- prompt: `/daily-todo`
