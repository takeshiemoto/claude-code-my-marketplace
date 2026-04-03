# claude-code-takeshiemoto-marketplace

Personal Claude Code plugin marketplace.

## Installation

### Via Claude Code CLI

```sh
claude plugin marketplace add takeshiemoto/claude-code-my-marketplace
claude plugin install takeshiemoto@takeshiemoto-marketplace
```

### Manual

Add to your `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "takeshiemoto-marketplace": {
      "source": {
        "source": "github",
        "repo": "takeshiemoto/claude-code-my-marketplace"
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
