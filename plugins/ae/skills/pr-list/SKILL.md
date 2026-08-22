---
name: pr-list
description: /pr-list - PR一覧表示
disable-model-invocation: true
argument-hint: '[open|closed|all]'
---

# /pr-list - PR一覧表示

Pull Requestの一覧を表示する。

**引数:** `/pr-list [open|closed|all]`（省略時は `open`）

## 手順

### 1. 引数の確認

$ARGUMENTS でstateフィルタを指定できる：

- `open`（デフォルト）: オープン中のPR
- `closed`: クローズ済みのPR
- `all`: すべてのPR

### 2. PR一覧の取得・表示

```bash
gh pr list --state {STATE}
```

結果をテーブル形式で見やすく表示する。
