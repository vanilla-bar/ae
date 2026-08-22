# ae

almost everywhere — 実装中のだいたいの作業をカバーする、Git/GitHub ワークフロー & 開発支援スキルプラグイン。

## 前提条件

- [GitHub CLI (`gh`)](https://cli.github.com/) — Issue/PR 操作系スキルで使用

## 開発フローとスキルの対応

```
作業開始
│
├─ /issue-open ··· やることを Issue に積む
├─ /issue-list ··· Issue 一覧を確認
│
▼
ブランチ作成
│
├─ /cut ·········· feature ブランチ + ドラフト PR を作成
├─ /cut-worktree · 同上（worktree 版。複数同時作成可）
│
▼
調査・設計
│
├─ /think ········ 要件ヒアリング + 批判的コード調査
│                  制約や前提の誤りを洗い出す
│
▼
実装
│
├─ /debug-mode ··· ログ計装による構造化デバッグ
│                  推測ではなく実行時証拠で原因を特定
├─ /commit ······· 変更を意味的なまとまりで分割コミット
├─ /self-review ·· サブエージェントによるセルフレビュー
│                  仕様書・計画書・コード差分を独立視点でチェック
│
▼
PR
│
├─ /pr-ready ····· ドラフト PR を更新してレビュー待ちに変更
├─ /pr-review ···· PR の差分を観点ベースでコードレビュー
├─ /pr-list ······ PR 一覧を確認
├─ /pr-merge ····· PR をマージしてローカルを整理
│
▼
整理
│
├─ /prune ········ マージ済みの不要ブランチを削除
│
▼
引き継ぎ（セッション終了時）
│
├─ /handover ····· 次のセッション向けの引き継ぎ書を生成
├─ /takeover ····· 引き継ぎ書を読み込んで作業を再開
```

## プロジェクト設定（任意）: `.claude/ae.config.md`

スキルはリポジトリ直下の `.claude/ae.config.md` があれば読み込み、GitHub Project 連携・CI チェック名・spec の置き場などをそこから取る。
**無くても全スキルは動く**（Project 連携や検証コマンドなど、設定に依存する手順がスキップされるだけ）。
プロジェクト固有の ID・メンバー名はプラグインではなくこのファイルに置く。

```markdown
# ae プロジェクト設定

## GitHub Project
- owner: my-org
- number: 7
- project_id: PVT_xxxxxxxx

| フィールド | フィールド ID | オプション |
| --- | --- | --- |
| Status | PVTSSF_xxxx | Todo=xxxxxxxx / In progress=xxxxxxxx / Done=xxxxxxxx |
| Priority | PVTSSF_xxxx | P0=xxxxxxxx / P1=xxxxxxxx / P2=xxxxxxxx / P3=xxxxxxxx / P4=xxxxxxxx |
| Estimate | PVTF_xxxx | Number |
| Iteration | PVTIF_xxxx | （動的） |
| Business | PVTSSF_xxxx | A=xxxxxxxx / B=xxxxxxxx / Other=xxxxxxxx |   <!-- 任意の単一選択フィールド。複数可 -->

## Issue
- labels: feature, bug, support, docs, refactor, research, chore
- assignees: alice, bob            <!-- 自分(@me) と 未アサイン は常に選択肢に入る -->
- template: .github/ISSUE_TEMPLATE/task.md

## Checks
- required_check: lint-test-build   <!-- /pr-merge が見る必須 CI チェック名 -->
- post_merge_check: pnpm exec contextlint
- spec_check: pnpm run lint:spec-integrity

## Docs
- spec_dir: docs/zones/{zone}/spec_{slug}.md
- adr_dir: docs/zones/{zone}/decisions/
- spec_template: docs/standards/spec-template.md
- adr_template: docs/standards/adr-template.md
- union_merge: docs/zones/*/decisions/README.md

## Backend
- backend_dir: amplify/

## Branches
- protected_branches: ^(master|dev|stage)$
```

フィールド ID は `gh project field-list {number} --owner {owner} --format json` で取得できる。
