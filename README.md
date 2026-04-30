# claude-code-takeshiemoto-marketplace

Personal Claude Code plugin marketplace.

## Installation

### Via Claude Code CLI

```sh
claude plugin marketplace add takeshiemoto/claude-code-my-marketplace
claude plugin install core@takeshiemoto-marketplace
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
    "core@takeshiemoto-marketplace": true
  }
}
```

## Uninstallation

```sh
claude plugin uninstall core@takeshiemoto-marketplace
claude plugin marketplace remove takeshiemoto-marketplace
```

## Plugins

| Plugin | Description |
| --- | --- |
| `core` | Personal toolkit |

### core:mo

Open a Markdown file in the browser via the `mo` command.

```sh
/core:mo <path>
```

### core:coderabbit-loop

Run an autonomous CodeRabbit feedback loop on a PR. Each round: collect findings → judge via `core:arbiter` subagent → patch (with type-check / lint gating) and commit → reply to rejected comments → re-review → check convergence.

```sh
/core:coderabbit-loop <PR#> [--base <branch>] [--max-rounds <N>] [--dry-run]
```

Persists state under `.coderabbit-loop/<run-id>/` and never pushes; the user decides when to push.

## Development

```sh
pnpm install
pnpm run lint:md
```
