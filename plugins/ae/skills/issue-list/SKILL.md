---
name: issue-list
description: GitHub Issue 一覧を表示。自分 or 未アサインのものを対象とし、プロジェクト設定に GitHub Project があれば Iteration グルーピング + Priority 順で表示する。ユーザーが「issue 一覧」「タスク一覧」「今のタスク」等と言った時にも使用する。
disable-model-invocation: true
argument-hint: '[open|closed|all]'
---

# /issue-list - Issue 一覧表示

GitHub Issue のうち、**自分にアサイン or 未アサイン**のものを、プロジェクト設定の GitHub Project に基づく **Iteration グルーピング + 各内 Priority 順**で表示する。設定が無ければ通常の一覧表示になる。

**引数:** `/issue-list [open|closed|all]`（省略時は `open`）

## プロジェクト設定の読み込み

リポジトリ直下の `.claude/ae.config.md` があれば読み込む（書式はプラグインの README「プロジェクト設定」を参照）。 `## GitHub Project` セクションから以下を取る:

- `{OWNER}` / `{PROJECT_NUMBER}`
- フィールド ID: Status（オプション ID 付き）/ Priority / Iteration

**設定ファイルが無い、または `## GitHub Project` が無い場合**は Project 連携をせず、手順 3 の結果を「番号 / タイトル / ラベル / アサイン」のテーブルで表示して終了する（手順 4〜7 はスキップ）。

ID が一致しない場合は `gh project field-list {PROJECT_NUMBER} --owner {OWNER} --format json` で再取得する。

## 手順

### 1. 引数の確認

$ARGUMENTS で state フィルタを指定する：

- `open`（デフォルト）: オープン中の Issue
- `closed`: クローズ済みの Issue
- `all`: すべての Issue

### 2. システム日付の取得

Iteration の Current/Next 判定に使う（AI の日付推定は当てにしない）:

```bash
TODAY=$(date +%Y-%m-%d)
```

PowerShell の場合:

```powershell
$TODAY = Get-Date -Format "yyyy-MM-dd"
```

### 3. Issue 一覧の取得

```bash
gh issue list --state {STATE} --limit 200 --json number,title,labels,assignees,state
```

### 4. Project アイテム + Iteration 設定の取得

```bash
gh project item-list {PROJECT_NUMBER} --owner {OWNER} --format json --limit 300
gh project field-list {PROJECT_NUMBER} --owner {OWNER} --format json --limit 50
```

field-list の Iteration field から `iterations: [{startDate, duration, title, id}, ...]` を取得。`TODAY` との関係で Current / Next を判定する:

- Current: `startDate <= TODAY < startDate + duration` を満たす Iteration
- Next: Current の次に始まる Iteration（Current が無い場合は `TODAY` 以降に始まる最初の Iteration）

### 5. フィルタ

Issue 一覧から以下を残す:

- 自分（`gh api user --jq .login` で取得）が assignees に含まれる
- もしくは assignees が空（未アサイン）

### 6. Project と Issue をマージ

Issue 番号と Project アイテムの `content.number` で突合し、各 Issue に Priority / Status / Iteration を付与する。Project に存在しない Issue はすべて未設定として扱う。

### 7. Iteration グルーピング + 各内 Priority 順で表示

Issue を以下のグループに振り分け、上から順に表示:

1. **Current Iteration**（{title} ({startDate}〜{endDate})）
2. **Next Iteration**（{title} ({startDate}〜{endDate})）
3. **その他の Iteration**（Current/Next 以外の Iteration ごとにグループ化、開始日昇順）
4. **バックログ**（Iteration 未割り当て）

各グループ内は Priority 昇順（P0→P4→未設定）、Priority が同じなら issue 番号昇順。

```
■ Current Iteration: Sprint #5 (2026-05-19 〜 2026-05-30)
  P0  #99  バグ調査: X が動かない                [todo]      @me
  P1  #56  GitHub Projects への移行              [in-prog]   @me
  P1  #55  Projects 操作スキルの追加              [todo]      (未アサイン)

■ Next Iteration: Sprint #6 (2026-06-02 〜 2026-06-13)
  P2  #54  issue-open スキルの改善               [in-prog]   @me

■ バックログ (Iteration 未割り当て)
  P3  #80  Y のリファクタ                          [todo]      @me
```

各行: `Priority  #番号  タイトル  [Status短縮]  アサイン情報`

- Priority 未設定の場合は `-` を表示
- Status 短縮: `todo` / `in-prog` / `done` / `-`（未設定）
- アサイン情報: 自分なら `@me`、未アサインなら `(未アサイン)`
- タイトルが端末幅を超える場合は末尾に `…` を付けて切り詰める
- 該当 Issue が無いグループは見出しごと省略する

### 8. 該当ゼロの場合

```
（該当 Issue はありません）
```

## 注意事項

- Issue は読み取り専用。フィールド更新は行わない
- `project` スコープ必須。未付与時は `gh auth refresh -s project` を案内
