# AGENTS.md

個人用 Claude Code プラグインマーケットプレイス。

## 開発コマンド

```sh
pnpm install
pnpm run lint:md
```

## アーキテクチャ

```text
.claude-plugin/marketplace.json   # マーケットプレイス定義
plugins/
  <plugin-name>/
    .claude-plugin/plugin.json    # プラグインメタデータ
    .mcp.json                     # MCP サーバー設定（任意）
    agents/
      <agent-name>.md             # サブエージェント定義
    skills/
      <skill-name>/
        SKILL.md                  # スキル定義（YAML frontmatter + 本文）
```

## バージョニング

- **プラグイン**: `plugins/<plugin-name>/.claude-plugin/plugin.json` の `version` を semver で管理
- **マーケットプレイス**: `.claude-plugin/marketplace.json` の `metadata.version` はプラグインの追加・削除時に上げる

## 新しいプラグインを追加する場合

1. `plugins/<plugin-name>/` ディレクトリを作成
2. `.claude-plugin/plugin.json` にメタデータを記述
3. 必要に応じて `.mcp.json`、`agents/*.md`、`skills/<skill-name>/SKILL.md` を追加
4. `.claude-plugin/marketplace.json` の `plugins` 配列に登録
