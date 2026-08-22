---
name: cut
description: featureブランチを作成しドラフトPRまで一気に作成する。ユーザーが「ブランチ切って」「ブランチ作って」「作業始めて」等と言った時にも使用する。
argument-hint: '[やりたいことの説明 or #issue番号]'
---

# /cut - featureブランチ作成 + ドラフトPR作成

GitHubのデフォルトブランチを起点にfeatureブランチを作成し、ドラフトPRまで一気に作成する。下記ルールに従えば確認なしで実行する。

**引数:** `/cut <やりたいことの説明>` または `/cut #123`（Issue番号指定）（省略時は対話で聞く）

## 手順

### 1. 引数の確認

$ARGUMENTS を確認し、以下のいずれかで処理を分岐する：

**A) Issue番号の場合**（`#123` や `123` のような数字のみのパターン）:

```bash
gh issue view {番号} --json title,labels,body
```

を実行し、Issueのタイトルをブランチの説明・PRタイトルとして使用する。ラベル情報はプレフィックス判定に活用する。本文（body）の「完了したら誰が何をできる / どうなるか」セクションは、step 4 で PR の `## 完了条件` に転記するため保持しておく。

**B) テキストの場合**: そのままブランチの説明・PRタイトルとして使用する。

**C) 引数なしの場合**: 以下のメッセージでユーザーに尋ねる：

```
やりたいことや実装したい機能を教えてください。
例: 「ユーザー一覧画面の追加」「ログイン画面のバグ修正」「#123」（Issue番号）
```

### 2. ブランチ名の自動生成

ブランチの説明（Issueタイトルまたはユーザー入力）から、以下のルールに従ってブランチ名を生成する：

**ラベル → プレフィックス対応表:**

| ラベル     | プレフィックス | 備考                                     |
| ---------- | -------------- | ---------------------------------------- |
| `feature`  | `feature/`     | 新規機能 / 機能拡張                      |
| `bug`      | `fix/`         | 不具合の調査〜修正                       |
| `refactor` | `refactor/`    | 振る舞いを変えずコード改善               |
| `docs`     | `docs/`        | リポジトリ内ドキュメントの場合のみ       |
| `chore`    | `chore/`       | 依存更新・CI・スクリプトなど雑務         |
| `research` | `chore/`       | コード変更を伴う技術検証の場合のみ       |
| `support`  | —              | cut 対象外（ブランチ不要なケースが多い） |

**テキスト入力時の判定:**

- 「○○の追加」「新機能」→ `feature/`
- 「バグ修正」「fix」→ `fix/`
- 「リファクタ」「整理」→ `refactor/`
- 「READMEの更新」「ドキュメント」→ `docs/`
- それ以外 → `chore/`

**Issue番号指定時の命名規則:**

- ブランチ名にIssue番号を含める（例: `feature/123-user-list`）
- ラベルから上の対応表でプレフィックスを判定する

**命名規則:**

- 小文字のみ使用
- 単語はハイフン（`-`）で区切る
- 簡潔で分かりやすい英語名
- 日本語は英語に変換

### 3. ブランチ作成 + 空コミット + push

```bash
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name')

git fetch origin
git checkout -b {BRANCH_NAME} origin/${DEFAULT_BRANCH}
git commit --allow-empty --no-verify -m "chore: start work on {説明}"
git push --no-verify -u origin {BRANCH_NAME}
```

### 4. ドラフトPR作成

**Issue番号指定時:**

step 1A で取得した Issue 本文から「完了したら誰が何をできる / どうなるか」セクションの内容を抽出し、PR 本文の `## 完了条件` に**チェックボックス形式で**転記する。完了条件は以後**不変の受け入れ基準**として扱い、`/pr-ready` はレビュー時にこのチェックボックスを確認（チェック）するだけで、項目の生成・追加はしない。

```bash
gh pr create --draft --base ${DEFAULT_BRANCH} --title "{Issueタイトル}" --body "$(cat <<'EOF'
Closes #{番号}

## 完了条件

- [ ] {Issue「完了したら誰が何をできる / どうなるか」の内容}
EOF
)"
```

Issue にこのセクションが無い（`task.md` テンプレ以前の Issue 等）場合は `## 完了条件` を省略し、`Closes #{番号}` のみとする。

**テキスト指定時:**

```bash
gh pr create --draft --base ${DEFAULT_BRANCH} --title "{説明テキスト}" --body "$(cat <<'EOF'
（作業中）
EOF
)"
```

### 5. 完了メッセージ

```
✅ ブランチ `{BRANCH_NAME}` を作成し、ドラフトPRを作成しました（origin/${DEFAULT_BRANCH}起点）
   PR: {PR_URL}
```

## 注意事項

このスキルの役割はブランチの作成とドラフトPR作成まで。完了メッセージを表示したら**そこで終了する**。
ブランチ作成後に自動で実装作業やファイル編集を開始してはならない。次の作業はユーザーの指示を待つこと。
