---
name: takeover
description: 現在ブランチの PR から引き継ぎコメントを読み込み、作業を再開する。
disable-model-invocation: true
argument-hint: '[PR番号（省略可）]'
---

# Takeover - 引き継ぎの読み込みと作業再開

## 注意事項

- コンテキスト汚染を防ぐため、読み込む対象が確定するまで引き継ぎコメントの本文を取得しないこと（ステップ1では本文を context に入れない）
- 引き継ぎコメントが存在しない場合は、その旨をユーザーに伝えること

## 手順

### ステップ1: 対象 PR と引き継ぎコメントの特定（本文は取得しない）

1. **対象 PR を特定する**:
   - $ARGUMENTS に PR 番号があればそれを対象にする
   - 無ければ現在ブランチの PR を対象にする

   ```bash
   gh pr view {PR番号があれば指定} --json number,headRefName
   ```

   PR が見つからない場合はユーザーに伝えて終了する。

2. **引き継ぎコメントを抽出する**（本文ではなくメタ情報のみを取得し、context を汚さない）:

   ```bash
   gh pr view {PR番号} --json comments \
     --jq '[.comments[] | select(.body | contains("<!-- claude-handover -->")) | {createdAt, url}] | sort_by(.createdAt) | last'
   ```

   - 該当が無ければ「引き継ぎコメントが見つかりません」と伝えて終了する
   - 得られた最新コメントの `url`（`...#issuecomment-{ID}`）から ID を取り出す

### ステップ2: 読み込みと文脈検証

3. **対象コメント本文を読み込む**（ファイルインデックスに記載されたファイル本体は読まない。タスク着手時に読めばよい）:

   ```bash
   OWNER_REPO=$(gh repo view --json owner,name --jq '"\(.owner.login)/\(.name)"')
   gh api "repos/$OWNER_REPO/issues/comments/{ID}" --jq .body
   ```

4. **文脈検証を行う**:
   - **ブランチ一致**: `git branch --show-current` と PR の `headRefName` を比較。不一致なら警告する。
   - **未コミット変更**: `git status --porcelain` が非空なら警告する。
   - **ドリフト**: 本文の `基準コミット` の SHA を取り出し、存在を確認する。

     ```bash
     git cat-file -e {SHA}^{commit}
     ```

     - **存在する場合**: `git diff --name-only {SHA}..HEAD` で変更ファイルを取得する。引き継ぎの「前提の完了状況」「コードに残っていない判断」が参照するファイル/領域と交差するものがあれば、その項目を名指しで「**この前提/判断は引き継ぎ以降に変更されている。着手前に再確認せよ**」と指示する。交差が無ければ触れない（ノイズを出さない）。
     - **存在しない場合**: 「基準コミットが見つかりません（履歴が書き換えられた可能性）。ドリフト判定は不可」と一言添える。

5. **報告する**: 「引き継ぎ読み込み完了」と伝える。警告（ブランチ不一致 / 未コミット変更 / ドリフト / 未完了の前提）があるときのみ列挙する。
