# claude-code-my-marketplace

Personal Claude Code plugin marketplace.

## Installation

### Via Claude Code CLI

```sh
/plugin marketplace add takeshiemoto/claude-code-my-marketplace
/plugin install takeshiemoto@takeshiemoto-claude-code-my-marketplace
```

### Manual

Add to your `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "my-marketplace": {
      "source": {
        "source": "github",
        "repo": "takeshiemoto/claude-code-my-marketplace"
      }
    }
  },
  "enabledPlugins": {
    "takeshiemoto@my-marketplace": true
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
