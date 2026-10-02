# cc-cmux-plugin

A [Claude Code plugin](https://code.claude.com/docs/en/plugins-reference) that integrates [cmux](https://cmux.com) — a native macOS terminal for AI coding agents built on the Ghostty engine — with Claude Code sessions.

## What it does

When installed, this plugin automatically:

- **Injects cmux context** into every Claude Code session — the full cmux command reference for status bar, pane management, browser control, and multi-agent orchestration
- **Routes notifications** through cmux's native notification system
- **Clears the progress bar** when Claude stops

Running state in the sidebar comes from cmux itself. Turn on the Claude Code integration in cmux Settings.

### Hooks

| Event | Behavior |
|-------|----------|
| `SessionStart` | Injects `rules/cmux-integration.md` into context |
| `Stop` | Clears the progress bar |
| `Notification` | Forwards Claude Code notifications to `cmux notify` |

Outside cmux (`$CMUX_WORKSPACE_ID` unset), every hook does nothing.

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
  plugin.json        # Plugin manifest and hook definitions
rules/
  cmux-integration.md  # Context injected into every session
skills/
  browser-automation/SKILL.md  # Browser pane workflows
```

## Requirements

- [cmux](https://cmux.com) 0.64.25 or later, with the session running inside a cmux workspace (`$CMUX_WORKSPACE_ID` set)
- Claude Code with plugin support
- `jq`, for the notification hook
