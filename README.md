# claude-code-my-marketplace

Personal Claude Code plugin marketplace.

## Installation

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
  }
}
```

Then enable the plugin:

```json
{
  "enabledPlugins": {
    "my@my-marketplace": true
  }
}
```

## Plugins

| Plugin | Description |
| --- | --- |
| `my` | Personal toolkit |

### my:review

Code review based on personal review guidelines.

```sh
/my:review --base <base-branch>
```

## Development

```sh
pnpm install
pnpm run lint:md
pnpm run lint:md:fix
```
