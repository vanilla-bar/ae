---
name: prune
description: /prune - 不要ブランチ削除
disable-model-invocation: true
---

# /prune - 不要ブランチ削除

マージ済み・リモート削除済みのローカルブランチおよびリモートブランチを整理する。

**実行前提:** worktree を削除するには、その worktree を掴んでいる Claude Code セッションが閉じている必要がある。worktree 内で開いたセッションのプロセスはディレクトリをロックし続け（cwd を移動しても解放されない）、セッションが生きている限り Windows のファイルロックで削除できない。**worktree を掴むセッションをすべて閉じてから、メイン側のセッションで実行すること。**

## 手順

### 1. リモート情報の更新

```bash
git fetch --prune
```

### 2. 削除候補の検出

**ローカルブランチ** — 以下の条件に該当するものを検出する：

- devにマージ済みのブランチ
- リモートが削除済み（gone）のブランチ

**リモートブランチ** — 以下の条件に該当するものを検出する：

- PRがマージ済み（closed/merged）でリモートに残っているブランチ

```bash
gh pr list --state merged --json headRefName --limit 100
git branch -r
git worktree list   # 各削除候補ブランチが worktree を持つか確認する
```

**絶対に削除しないブランチ:** デフォルトブランチと、プロジェクト設定 `protected_branches` に一致するもの（既定 `main` / `master` / `develop` / `dev`。`stage` などがあれば設定に足す）

### 3. 削除候補の表示

AskUserQuestionツールを使い、削除候補をユーザーに提示して確認する：

```
## 削除候補のブランチ

### ローカル
| ブランチ名 | 状態 | worktree |
|-----------|------|----------|
| feature/xxx | マージ済み | あり（../worktrees/feature/xxx） |
| fix/yyy | リモート削除済み | なし |

### リモート
| ブランチ名 | 状態 |
|-----------|------|
| origin/feature/zzz | PRマージ済み |

上記のブランチを削除してよいですか？
```

削除候補がない場合はその旨を伝えて終了する。

### 4. 削除実行

ユーザーの承認後、各ブランチを削除する：

**ローカル:**

各ブランチについて worktree の有無を確認し、ある場合は二段階で削除してからブランチを消す。

```bash
WT_PATH=$(git worktree list --porcelain | grep -B2 "branch refs/heads/{BRANCH_NAME}$" | head -1 | sed 's/^worktree //')

if [ -n "$WT_PATH" ]; then
  # worktree あり: ① node_modules を削除（Windows では pnpm の NTFS junction 対策で rmdir を使う）
  if [ -d "$WT_PATH/node_modules" ]; then
    case "$(uname -s)" in
      MINGW*|MSYS*|CYGWIN*) cmd //c "rmdir /S /Q \"$(cygpath -w "$WT_PATH/node_modules")\"" ;;
      *) rm -rf "$WT_PATH/node_modules" ;;
    esac
  fi
  # ② worktree 本体を削除してからブランチ削除
  git worktree remove "$WT_PATH" --force
  git branch -D {BRANCH_NAME}
else
  # worktree なし: 通常削除
  git branch -d {BRANCH_NAME}
fi
```

`-d` で削除できない場合（未マージ）はスキップし、ユーザーに報告する。

`git worktree remove` が `being used by another process` で失敗した場合は、その worktree をまだ掴んでいるセッションが残っている。リトライせず、対象 worktree（`$WT_PATH`）を開いている作業セッションを閉じてから `/prune` を再実行するよう案内し、そのブランチはスキップする。

全ブランチの処理後、参照切れの worktree を掃除する:

```bash
git worktree prune
```

**リモート:**

```bash
git push origin --delete {BRANCH_NAME}
```

### 5. 完了報告

削除結果を報告する。
