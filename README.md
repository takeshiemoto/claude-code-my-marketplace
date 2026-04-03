# claude-code-takeshiemoto-marketplace

Personal Claude Code plugin marketplace.

## Installation

### Via Claude Code CLI

```sh
/plugin marketplace add takeshiemoto/claude-code-takeshiemoto-marketplace
/plugin install takeshiemoto@takeshiemoto-claude-code-takeshiemoto-marketplace
```

### Manual

Add to your `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "takeshiemoto-marketplace": {
      "source": {
        "source": "github",
        "repo": "takeshiemoto/claude-code-takeshiemoto-marketplace"
      }
    }
  },
  "enabledPlugins": {
    "takeshiemoto@takeshiemoto-marketplace": true
  }
}
```

## Plugins

| Plugin | Description |
| --- | --- |
| `takeshiemoto` | Personal toolkit |

### takeshiemoto:review

Parallel code review with specialized sub-agents.

```sh
/takeshiemoto:review --base <base-branch>
```

## Development

```sh
pnpm install
pnpm run lint:md
```
