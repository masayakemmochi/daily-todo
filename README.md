# daily-todo

Slack と Jira から「今日自分がやるべきこと」を集約し、自分宛の Slack DM にサマリを送る Claude Code スキル。

## 前提

- [Claude Code](https://claude.com/claude-code)
- Slack MCP サーバー（`mcp__slack__*` のツール群）
- Atlassian MCP サーバー（`mcp__atlassian__jira_search`）

どちらかが使えない場合も処理は止まらず、サマリ末尾に「取得失敗」として記載される。

## 導入

```sh
git clone https://github.com/masayakemmochi/daily-todo.git
ln -s "$(pwd)/daily-todo" ~/.claude/skills/daily-todo
cp daily-todo/config.example.md daily-todo/config.md
```

`config.md` が無い状態で起動した場合も、スキルが `config.example.md` から自動で作る。

## 設定

`config.md` に記載する。`SLACK_USER_ID` と `SLACK_SELF_DM_ID` は空欄のままでよい。初回実行時に Slack から取得して書き込む。

| 項目 | 意味 | 既定値 |
|---|---|---|
| `SLACK_USER_ID` | 自分の Slack ユーザー ID | 初回に自動取得 |
| `SLACK_SELF_DM_ID` | 自分宛 DM のチャンネル ID（サマリの送信先） | 初回に自動取得 |
| `WORKDAY_END` | 収集期間の起点となる終業時刻 | `18:00` |
| `DUE_SOON_DAYS` | 「期限が近い」とみなす日数 | `3` |
| `TIMEZONE` | タイムゾーン | `Asia/Tokyo` |

収集期間は、前営業日の `WORKDAY_END` から実行時点までである（月曜は前週金曜から）。

## 使い方

Claude Code で次のいずれかを入力する。

- `/daily-todo`
- 「今日やること」「朝のサマリ」「デイリーTODO」

「毎朝自動で」と依頼すると、平日 9:00 に実行する定時ジョブを `CronCreate` で登録する。この登録は Claude Code のセッションが起動している間だけ有効である。

## 出力例

```
📋 2026-11-19(木) 今日やること

🔴 今日やる（2件）
• PROJ-123 ログイン不具合の修正をレビューする（期限: 11/19）
• #dev @相手 API 仕様の確認依頼に回答する

🟡 返信・確認待ち（1件）
• PROJ-98 コメントでの質問に返信する

🔵 期限が近い（1件）
• PROJ-110 設計書を更新する（期限: 11/21）

⚪ 参考（3件）
• ...
```

各項目には Slack のパーマリンク、または Jira 課題へのリンクが付く。

## 注意点

- 送信先は `SLACK_SELF_DM_ID`（自分宛 DM）に限定している。他者への誤送信を防ぐためである。
- `config.md` は個人の ID を含むため、`.gitignore` で git 管理外にしている。
