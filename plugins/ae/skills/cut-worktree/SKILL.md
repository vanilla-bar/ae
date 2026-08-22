---
name: cut-worktree
description: /cut-worktree - featureブランチ作成（worktree）+ ドラフトPR作成。複数指定でまとめて作成可能。
disable-model-invocation: true
argument-hint: '[#issue番号やテキスト（複数可）]'
---

# /cut-worktree - featureブランチ作成（worktree）+ ドラフトPR作成

GitHubのデフォルトブランチを起点にfeatureブランチをworktreeとして作成し、ドラフトPRまで一気に作成する。
メインの作業ディレクトリを切り替えずに並行作業したい場合に使う。複数指定でまとめて作成可能。下記ルールに従えば確認なしで実行する。

**引数の例:**

- `/cut-worktree #123` — 1つ
- `/cut-worktree #123, #124` — 複数（カンマ区切り）
- `/cut-worktree ログイン画面の修正` — 1つ
- `/cut-worktree #123, ログイン画面の修正` — Issue + テキスト混在
- `/cut-worktree ログイン修正, CSV追加` — テキスト複数
- 省略時は対話で聞く

## 手順

### 1. 引数のパース

$ARGUMENTS をカンマ（`,`）で分割し、**作成項目リスト**を作る：

**パース規則:**

- カンマ（`,`）で区切って項目に分割する（1つだけならカンマなしでOK）
- 各項目の前後の空白をトリムする
- `#` + 数字のパターン → Issue番号
- それ以外 → テキスト（そのままブランチの説明として使用）
- 引数なしの場合は対話で聞く

**パース例:**

| 入力                       | 作成項目リスト                                   |
| -------------------------- | ------------------------------------------------ |
| `#123`                     | [Issue #123]                                     |
| `#123, #124`               | [Issue #123, Issue #124]                         |
| `ログイン画面の修正`       | [テキスト: ログイン画面の修正]                   |
| `ログイン修正, CSV追加`    | [テキスト: ログイン修正, テキスト: CSV追加]      |
| `#123, ログイン画面の修正` | [Issue #123, テキスト: ログイン画面の修正]       |
| `#123, ログイン修正, #124` | [Issue #123, テキスト: ログイン修正, Issue #124] |

### 2. Issue情報の取得

作成項目リストにIssue番号が含まれる場合、各Issueの情報を取得する：

```bash
gh issue view {番号} --json title,labels,body
```

Issueのタイトルをブランチの説明・PRタイトルとして使用する。ラベル情報はプレフィックス判定に活用する。本文（body）の「完了したら誰が何をできる / どうなるか」セクションは、step 4 で PR の `## 完了条件` に転記するため保持しておく。

### 3. ブランチ名の自動生成

各項目について、以下のルールに従ってブランチ名を生成する：

**ラベル → プレフィックス対応表:**

| ラベル     | プレフィックス | 備考                                              |
| ---------- | -------------- | ------------------------------------------------- |
| `feature`  | `feature/`     | 新規機能 / 機能拡張                               |
| `bug`      | `fix/`         | 不具合の調査〜修正                                |
| `refactor` | `refactor/`    | 振る舞いを変えずコード改善                        |
| `docs`     | `docs/`        | リポジトリ内ドキュメントの場合のみ                |
| `chore`    | `chore/`       | 依存更新・CI・スクリプトなど雑務                  |
| `research` | `chore/`       | コード変更を伴う技術検証の場合のみ                |
| `support`  | —              | cut-worktree 対象外（ブランチ不要なケースが多い） |

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

### 4. worktree作成 + 空コミット + push + ドラフトPR

各項目に対して以下を順番に実行する：

```bash
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name')
git fetch origin  # 初回のみ

# --- 各項目について繰り返し ---

# worktree作成
git worktree add -b {BRANCH_NAME} ../worktrees/{BRANCH_NAME} origin/${DEFAULT_BRANCH}

# 環境ファイルコピー（gitignore 対象の .env*/amplify_outputs* を自動追従。スキル追記不要）
git ls-files --others --ignored --exclude-standard \
  | grep -E '^(\.env|amplify_outputs)' \
  | while read -r f; do
      cp "$f" "../worktrees/{BRANCH_NAME}/$f"
    done

# 依存インストール（lockfile からパッケージマネージャを判定）
cd ../worktrees/{BRANCH_NAME}
if [ -f pnpm-lock.yaml ]; then pnpm install
elif [ -f yarn.lock ]; then yarn install
elif [ -f bun.lock ] || [ -f bun.lockb ]; then bun install
elif [ -f package-lock.json ]; then npm install
fi

# 空コミット + push
git commit --allow-empty --no-verify -m "chore: start work on {説明}"
git push --no-verify -u origin {BRANCH_NAME}

# ドラフトPR作成
gh pr create --draft --base ${DEFAULT_BRANCH} --title "{タイトル}" --body "$(cat <<'EOF'
{本文}
EOF
)"
```

**PR本文の決定:**

- Issue由来の項目: `Closes #{番号}`。さらに step 2 で取得した Issue 本文に「完了したら誰が何をできる / どうなるか」セクションがあれば、その内容を `## 完了条件` に**チェックボックス形式で**転記する（不変の受け入れ基準。`/pr-ready` がチェックする）。セクションが無い Issue は `Closes #{番号}` のみ
- テキスト由来の項目: `（作業中）`

Issue由来の項目の本文は以下の形になる：

```markdown
Closes #{番号}

## 完了条件

- [ ] {Issue「完了したら誰が何をできる / どうなるか」の内容}
```

### 5. 完了メッセージ

`pwd` で取得した絶対パスをもとに、cdコマンドを絶対パスで案内する。

**1項目の場合:**

```
✅ Worktree `{BRANCH_NAME}` を作成し、ドラフトPRを作成しました（origin/${DEFAULT_BRANCH}起点）

  PR: {PR_URL}

新しいターミナルで以下を実行してください：
  cd {WORKTREE_ABSOLUTE_PATH}
```

**2項目以上の場合:**

```
✅ {N}個のworktreeを作成しました（origin/${DEFAULT_BRANCH}起点）

  1. feature/123-user-list  PR: {PR_URL_1}
     cd {WORKTREE_1_ABSOLUTE_PATH}

  2. feature/login-fix      PR: {PR_URL_2}
     cd {WORKTREE_2_ABSOLUTE_PATH}
```

## 注意事項

このスキルの役割はworktreeの作成とドラフトPR作成まで。完了メッセージを表示したら**そこで終了する**。
ブランチ作成後に自動で実装作業やファイル編集を開始してはならない。次の作業はユーザーの指示を待つこと。
