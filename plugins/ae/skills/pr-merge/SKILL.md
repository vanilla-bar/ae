---
name: pr-merge
description: /pr-merge - PRマージ
disable-model-invocation: true
argument-hint: '[PR番号]'
---

# /pr-merge - PRマージ

指定された Pull Request をマージする。マージ方式は merge commit。

**引数:** `/pr-merge [PR番号]`（省略時は現在のブランチのPR）

手順 1 で PR の状態を判定し、以下のいずれか 1 経路を実行する。

- 直ちにマージ可能（`CLEAN`）→ 手順 3 で**その場でマージ**する。
- CI 保留中 → 手順 3' で **auto-merge を予約し、CI 完了 → マージ確定まで見届けて**ローカルの片付けまで行う（CI 以外の要件待ち＝ master のレビュー承認などが残る場合は、予約したまま報告して終了）。
- CI 失敗・未解決のコンフリクト・ドラフト → **報告して停止**する（マージしない）。

## プロジェクト設定

リポジトリ直下の `.claude/ae.config.md` があれば読み込む（書式はプラグインの README を参照）。

- `required_check`: 必須 CI チェック名（未設定なら `statusCheckRollup` の全要素を対象にする）
- `union_merge` / `post_merge_check`: `/pr-ready` と同じ
- `protected_branches`: 保護ブランチの正規表現（既定 `^(main|master|develop|dev)$`）

## 手順

### 1. 対象PRの特定

`$ARGUMENTS` でPR番号が指定されていればそのPRを対象にする。指定がなければ現在のブランチに関連するPRを対象にする。

```bash
gh pr view {PR_NUMBER} --json number,title,state,mergeable,mergeStateStatus,statusCheckRollup,autoMergeRequest,url,headRefName,baseRefName
```

- PRが存在しない場合はその旨を伝えて終了する。
- `state` が `MERGED` の場合 → 手順2（保護判定）を実行し、手順3・手順3'を飛ばして手順4（ローカルの整理）へ進み、完了報告する。
- それ以外は `mergeStateStatus` で次の経路に分岐する。**自分でブロック要因を解消しようとしない**（下表の指定以外の操作をしない）。

| mergeStateStatus     | 経路                                                                 |
| -------------------- | -------------------------------------------------------------------- |
| `CLEAN`              | 手順2（保護判定）→ 手順3（直接マージ）                               |
| `BLOCKED`/`UNSTABLE` | 手順1a（必須チェックの状態で分岐）                                   |
| `BEHIND`             | `gh pr update-branch {PR_NUMBER}` で base を取り込む旨を案内して終了 |
| `DIRTY`              | 手順1b（コンフリクト処理）                                           |
| `DRAFT`              | `/pr-ready` で ready にしてから再実行する旨を案内して終了            |
| `UNKNOWN`            | 数秒待って手順1を最初からやり直す                                    |

### 1a. `BLOCKED`/`UNSTABLE` のとき

`statusCheckRollup` から必須チェック（`.name` または `.context` が `required_check` に一致する要素。未設定なら全要素）を取り出し、その状態で分岐する。

- `conclusion` が `FAILURE` / `TIMED_OUT` / `CANCELLED` / `ACTION_REQUIRED` / `STARTUP_FAILURE` のいずれか（＝失敗）
  → 失敗したチェック名と、ログの参照先（PR の Checks タブ / `url`）を報告して終了する。**マージも予約もしない。**
- 上記以外（`status` が `IN_PROGRESS` / `QUEUED` / `PENDING` で `conclusion` が空、または `conclusion` が `SUCCESS`）
  → 手順2（保護判定）→ 手順3'（auto-merge 予約）へ進む。

### 1b. `DIRTY`（コンフリクト）のとき

head ブランチをチェックアウトした状態（通常は現在のブランチ）で次を実行する。

```bash
git fetch origin
git merge origin/{BASE_REF}            # union_merge 対象は自動解消される
git diff --name-only --diff-filter=U   # 未解決の衝突ファイル一覧
```

- `git diff --name-only --diff-filter=U` の出力が**空**の場合:
  1. `post_merge_check` があれば実行する（無ければ 2 へ）。
  2. パスしたら `git push` し、手順1を最初からやり直す。
  3. `post_merge_check` が失敗したら `git merge --abort` し、下記の「報告して終了」に切り替える。
- 出力に **`union_merge` 以外**のファイルが含まれる、または head ブランチをローカルにチェックアウトできない場合:
  - `git merge` を実行済みなら `git merge --abort` する。
  - 次を報告して終了する（**自分で解消しない**）:
    - コンフリクトが発生しているファイル
    - 何と何が競合しているか（どのブランチのどの変更同士か）
    - 解消の方針案

### 2. head ブランチが保護ブランチか判定

プロジェクト設定の `protected_branches` で判定する（リポに `.githooks/pre-push` 等の保護がある場合は同じ値にしておく）。

```bash
PROTECTED_BRANCHES="{protected_branches}"   # 既定: ^(main|master|develop|dev)$

if [[ "{HEAD_REF}" =~ $PROTECTED_BRANCHES ]]; then
  IS_PROTECTED=true
else
  IS_PROTECTED=false
fi
```

保護ブランチを増やす場合はプロジェクト設定の `protected_branches` と、リポ側の pre-push フック等を同じ値に揃える。

### 3. 直接マージ（`CLEAN` のとき）

まず PR をマージする。**`--delete-branch` は付けない。**

```bash
gh pr merge {PR_NUMBER} --merge
```

- `--delete-branch` を付けないのは、それがマージ後にローカルで `git checkout {base}` を試み、base が別 worktree で使用中だと**失敗し、リモート削除ごと巻き添えにする**ため（worktree 運用で頻発する）。リモート削除は下で明示的に行う。
- マージ後、`gh pr view {PR_NUMBER} --json state --jq .state` が `MERGED` であることを確認する。
- `IS_PROTECTED=false`（フィーチャーブランチ）の場合のみ、リモートブランチを削除する（ローカルの checkout を伴わないので worktree に依存しない）。保護ブランチ（`IS_PROTECTED=true`）では削除しない。

```bash
git push origin --delete {HEAD_REF}
```

- マージ完了後、手順4（ローカルの整理）へ進む。

### 3'. auto-merge 予約とマージの見届け（`BLOCKED`/`UNSTABLE` の保留時）

**予約 → CI 完了まで待機 → マージ確定を確認 → 手順4（片付け）まで、この起動の中で見届ける。**

#### 3'-1. auto-merge を予約

`IS_PROTECTED` の値で削除フラグを切り替えて `--auto` で予約する。

```bash
if [ "$IS_PROTECTED" = "true" ]; then
  gh pr merge {PR_NUMBER} --auto --merge
else
  gh pr merge {PR_NUMBER} --auto --merge --delete-branch
fi
```

- コマンドが "clean status" 等のエラーで弾かれた場合は、手順1を最初からやり直す（`CLEAN` になっていれば手順3で直接マージされる）。
- 予約が成立したら 3'-2 へ進む。

#### 3'-2. CI 完了まで待機

必須チェックが終わるまで待つ。

```bash
gh pr checks {PR_NUMBER} --watch --required --fail-fast
```

- 「no checks reported」で即座に終了した場合（予約直後で CI が未登録）は、数秒待って 1 回だけ再実行する。
- 待機が実行環境のタイムアウトで打ち切られた場合は、同じ `--watch` を再実行して待機を続ける（冪等。予約は 3'-1 で済んでいるので何度実行してもよい）。
- 終了コードで分岐する:
  - `0`（必須チェックが全て緑）→ 3'-3 へ進む。
  - それ以外（必須チェックが失敗）→ 手順5'（失敗・保留の報告）へ。**auto-merge の予約は解除しない**（修正を push すれば緑化後に自動マージされるため）。

#### 3'-3. マージ確定を確認

auto-merge は必須チェックが緑になった直後に走る。`state` が `MERGED` になるまで数秒間隔で最大 ~30 秒確認する。

```bash
gh pr view {PR_NUMBER} --json state,mergeStateStatus --jq '{state, mergeStateStatus}'
```

- `state == MERGED` → 手順4（ローカルの整理）→ 手順5（完了報告）へ進む。
- ~30 秒待っても `MERGED` にならない場合（CI 以外の未充足要件が残っている。例: master のレビュー承認待ち）→ 手順5'（保留の報告）へ。**予約は解除しない**（要件充足後に GitHub が自動マージする）。

### 4. ローカルの整理（直接マージ後 / 既に `MERGED` のとき）

`IS_PROTECTED=true`（保護ブランチ）の場合は、デフォルトブランチを最新化して**終了する**（ブランチ・worktree の削除はしない）。

```bash
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name')
git checkout ${DEFAULT_BRANCH}
git pull origin ${DEFAULT_BRANCH}
```

以降は `IS_PROTECTED=false`（フィーチャーブランチ）のときのみ実行する。

#### 4a. head ブランチの worktree 判定

```bash
# HEAD_REF がチェックアウトされている worktree のパス（無ければ空）
WT_PATH=$(git worktree list --porcelain | grep -B2 "branch refs/heads/{HEAD_REF}$" | head -1 | sed 's/^worktree //')

# メイン作業ツリー（worktree list の先頭）で HEAD_REF がチェックアウトされている場合は、
# 専用のリンク worktree ではないので 4c で通常削除する。
# 比較対象は「現在地(show-toplevel)」ではなく「メイン worktree」であることに注意:
# head 専用のリンク worktree の中から実行した場合、現在地と一致してしまい 4c(checkout 失敗)へ
# 誤誘導されるため。リンク worktree にいる場合は 4b で /prune に委ねる。
MAIN_WT=$(git worktree list --porcelain | head -1 | sed 's/^worktree //')
if [ "$WT_PATH" = "$MAIN_WT" ]; then
  WT_PATH=""
fi
```

- `WT_PATH` が空 → 手順4c へ
- `WT_PATH` に値あり → 手順4b へ

#### 4b. worktree がある場合

worktree もローカルブランチも削除せず、以下を手順5の完了報告にそのまま含めて終了する（リモート削除は手順3で完了済み）。

```
worktree が残っています（{WT_PATH}）。
worktree 内で開いている作業セッションを閉じてから、メイン側のセッションで /prune を実行してください。
リモートブランチは削除済みのため、/prune が「リモート削除済み(gone)」として worktree ごと回収します。
```

#### 4c. デフォルトブランチの最新化とローカルブランチ削除（worktree なしのとき）

```bash
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name')
git checkout ${DEFAULT_BRANCH}
git pull origin ${DEFAULT_BRANCH}
git branch -D {HEAD_REF} 2>/dev/null || true
```

### 5. 完了報告（直接マージ / 既にマージ済み）

以下を伝える。

- マージ完了。
- head ブランチのリモート削除有無（`IS_PROTECTED` の値）。
- worktree が残っている場合（手順4b）は、その案内文をそのまま含める。
- worktree なしのフィーチャーブランチ（手順4c）でローカルブランチを削除した場合はその旨。

### 5'. 保留・失敗の報告（見届け中にマージまで到達しなかったとき）

手順3'-2 で必須チェックが失敗した場合、または手順3'-3 で待っても `MERGED` に到達しなかった場合に、以下を伝えて終了する。auto-merge の予約は解除しない。

- **必須チェックが失敗したとき**: 失敗したチェック名とログの参照先（PR の Checks タブ / `url`）。auto-merge は予約したままなので、**修正を push すれば緑化後に自動マージ**されること。
- **CI 以外の要件待ちのとき（例: master のレビュー承認）**: 何を待っているか。要件充足後に GitHub が自動でマージし、フィーチャーブランチならリモートブランチも自動削除すること。
- いずれの場合も、マージ結果の確認とローカルの後片付けは、後で `/pr-merge {PR_NUMBER}` を再実行すれば行えること（マージ済みなら手順4のローカル整理まで実施される）。worktree が残っている場合は `/prune` で回収すること。
