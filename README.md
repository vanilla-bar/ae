# kernel

Claude Code 向けのプラグインマーケットプレイス。

## インストール

### npx skills（推奨）

```bash
# 全スキルを一括インストール
npx skills add vanilla-bar/kernel --skill "*" -a claude-code --copy

# グローバル（全プロジェクト共通）にインストール
npx skills add vanilla-bar/kernel --skill "*" -a claude-code --copy -g
```

### アップデート

```bash
npx skills check
npx skills update
```

### プラグイン

```bash
# 1. マーケットプレイスを追加
/plugin marketplace add vanilla-bar/kernel

# 2. プラグインをインストール
/plugin install ae@kernel
```

> **Note:** プラグイン方式はセッション間でスキルが消える既知の問題があります（[#17089](https://github.com/anthropics/claude-code/issues/17089)）。現時点では `npx skills` を推奨します。

## プラグイン一覧

| プラグイン | 説明 |
|-----------|------|
| [ae](plugins/ae/) | Git/GitHub ワークフロー & 開発支援スキル集（15スキル。v0.2.0 で `.claude/ae.config.md` によるプロジェクト設定に対応） |

## ライセンス

[MIT](LICENSE)
