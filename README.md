# cc-cmux-plugin

A [Claude Code plugin](https://code.claude.com/docs/en/plugins-reference) that integrates [cmux](https://cmux.dev) — a native macOS terminal for AI coding agents built on the Ghostty engine — with Claude Code sessions.

## What it does

When installed, this plugin automatically:

- **Injects cmux context** into every Claude Code session — the full cmux command reference for status bar, pane management, browser control, and multi-agent orchestration
- **Sets agent status** in the cmux sidebar (`Active` on session start, `Idle` on stop)
- **Routes notifications** through cmux's native notification system
- **Grants permissions** for all `cmux` CLI commands

### Hooks

| Event | Behavior |
|-------|----------|
| `SessionStart` | Injects `rules/cmux-integration.md` into context; sets status to "Active" |
| `Stop` | Sets status to "Idle"; clears progress bar |
| `Notification` | Forwards Claude Code notifications to `cmux notify` |

## Installation

### Marketplace (recommended)

First, register the marketplace:

```bash
/plugin marketplace add KyleJamesWalker/cc-cmux-plugin
```

Then install the plugin:

```bash
claude plugin install cmux-integration@KyleJamesWalker-cc-cmux-plugin
```

You can also register and enable it manually in `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "KyleJamesWalker-cc-cmux-plugin": {
      "source": {
        "source": "github",
        "repo": "KyleJamesWalker/cc-cmux-plugin"
      }
    }
  },
  "enabledPlugins": {
    "cmux-integration@KyleJamesWalker-cc-cmux-plugin": true
  }
}
```

### Local development

Load the plugin directly from a local checkout (for the current session only):

```bash
git clone https://github.com/KyleJamesWalker/cc-cmux-plugin.git
claude --plugin-dir ./cc-cmux-plugin
```

## Updating

Plugins are cached locally and keyed by the `version` field in `plugin.json`. To pick up new changes:

```bash
claude plugin update cmux-integration@KyleJamesWalker-cc-cmux-plugin
```

## Project structure

```
.claude-plugin/
  marketplace.json   # Marketplace registry metadata
  plugin.json        # Plugin manifest (name, version, permissions, hooks ref)
hooks/
  hooks.json         # Hook definitions (SessionStart, Stop, Notification)
rules/
  cmux-integration.md  # Context injected into every session
```

## Requirements

- [cmux](https://cmux.dev) must be installed and the session must be running inside a cmux workspace (`$CMUX_WORKSPACE_ID` set)
- Claude Code with plugin support
