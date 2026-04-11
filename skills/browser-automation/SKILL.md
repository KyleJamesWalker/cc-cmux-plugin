---
name: browser-automation
description: Use when the user asks to open websites, search the web, scrape or extract data from web pages, take screenshots, fill forms, or interact with web content through cmux browser panes. Also use when automating any multi-step browser workflow.
---

# Browser Automation in cmux

Automate web browsing tasks using cmux browser panes — navigation, searching, data extraction, screenshots, and page interaction.

## Prerequisites

- Must be running inside cmux (`$CMUX_WORKSPACE_ID` is set)
- Browser commands follow: `cmux browser [--surface <ref>] <subcommand> [args]`
- The `--surface` flag goes BEFORE the subcommand — placing it after will error

## Workflow

### 1. Open a Browser Pane

Open a browser pane and capture the `surface:N` reference for all subsequent commands:

```bash
cmux new-pane --type browser --direction right --url "https://example.com"
# => OK surface:35 pane:29 workspace:3
# Store the surface:N value — you need it for every command below.
```

Use `--direction` to control placement: `right`, `left`, `up`, `down`.

### 2. Wait for Page Load, Then Inspect

After any `navigate`/`goto`, the page needs time to render. Use a short delay (under 2 seconds) then `snapshot --compact` to inspect the accessibility tree:

```bash
sleep 1.5 && cmux browser --surface surface:N snapshot --compact
```

- `--compact` keeps output manageable for large pages
- `--selector <css>` scopes the snapshot to a specific area
- `-i` (interactive) adds `ref=eNNN` identifiers you can target with click/type/fill

### 3. Navigate and Search

**Prefer URL navigation over form interaction for search engines.** Google, Craigslist, and other sites with autocomplete/combobox widgets frequently throw JS exceptions when automated via `click`/`type`/`fill`.

```bash
# RELIABLE — construct the search URL directly
cmux browser --surface surface:N navigate "https://www.google.com/search?q=search+query"
cmux browser --surface surface:N navigate "https://losangeles.craigslist.org/search/sss?query=dressers"
cmux browser --surface surface:N navigate "https://www.amazon.com/s?k=search+terms"

# FRAGILE — form interaction on complex widgets often fails
cmux browser --surface surface:N click "ref=e591"    # JS exception on comboboxes
cmux browser --surface surface:N type "ref=e591" "query"  # JS exception
```

For sites without URL-based search, use the interaction fallback order in section 6.

### 4. Extract Data

**Use `eval` for JavaScript execution** (not `js`, `evaluate`, or `exec`):

```bash
# Simple values
cmux browser --surface surface:N eval "document.title"
cmux browser --surface surface:N eval "document.querySelectorAll('.item').length"

# Structured data — pipe JSON directly to a file
cmux browser --surface surface:N eval "JSON.stringify(
  Array.from(document.querySelectorAll('a.titlestring')).map(a => ({
    title: a.textContent.trim(),
    link: a.href
  })), null, 2)" > results.json
```

**Use `get` for simple extractions without JavaScript:**

```bash
cmux browser --surface surface:N get url                        # current URL
cmux browser --surface surface:N get title                      # page title
cmux browser --surface surface:N get text --selector "h1"       # element text content
cmux browser --surface surface:N get html --selector ".results" # element innerHTML
cmux browser --surface surface:N get count --selector "li.item" # count matching elements
cmux browser --surface surface:N get attr --selector "a" --attr "href"  # attribute value
```

### 5. Take Screenshots

```bash
cmux browser --surface surface:N screenshot --out /path/to/output.jpg
```

To capture content below the fold, scroll first:

```bash
cmux browser --surface surface:N scroll --dy 500
cmux browser --surface surface:N screenshot --out /path/to/below-fold.jpg
```

### 6. Interact with Page Elements

**Fallback order** — try each approach in order when the previous one fails:

| Priority | Method | When to use |
|----------|--------|-------------|
| 1st | `navigate` with URL params | Search engines, filtered listings, any URL-driven UI |
| 2nd | `fill` on selector | Simple input fields, text areas |
| 3rd | `click` then `type` | Buttons, interactive widgets |
| 4th | `eval` with DOM manipulation | Last resort when all interaction commands fail |

**Interaction commands:**

```bash
# Click / hover / focus
cmux browser --surface surface:N click "ref=e123"
cmux browser --surface surface:N click --selector "button.submit"
cmux browser --surface surface:N hover --selector ".dropdown-trigger"

# Fill input fields (sets value programmatically)
cmux browser --surface surface:N fill "ref=e456" "text to enter"
cmux browser --surface surface:N fill --selector "input[name=email]" --text "user@example.com"

# Type (simulates keystrokes — use for widgets that need key events)
cmux browser --surface surface:N type "ref=e456" "text to type"

# Press keys
cmux browser --surface surface:N press "Enter"
cmux browser --surface surface:N press "Tab"

# Select dropdowns
cmux browser --surface surface:N select --selector "select#country" --value "US"

# Checkboxes
cmux browser --surface surface:N check --selector "input#agree"
cmux browser --surface surface:N uncheck --selector "input#newsletter"
```

Use `--snapshot-after` on any interaction command to get the page state after the action in a single call.

### 7. Wait for Dynamic Content

```bash
# Wait for a selector to appear
cmux browser --surface surface:N wait --selector ".results-loaded"

# Wait for specific text
cmux browser --surface surface:N wait --text "Search complete"

# Wait for URL change
cmux browser --surface surface:N wait --url-contains "/results"

# Wait for page load state
cmux browser --surface surface:N wait --load-state "complete"

# Custom JS condition
cmux browser --surface surface:N wait --function "document.querySelectorAll('.item').length > 0"

# With timeout
cmux browser --surface surface:N wait --selector ".slow-content" --timeout 30
```

## Common Mistakes

- **Placing `--surface` after the subcommand** — it must come before: `cmux browser --surface surface:N navigate`, not `cmux browser navigate --surface surface:N`
- **Using `js`/`evaluate`/`exec` instead of `eval`** — only `eval` is a valid subcommand
- **Trying to automate Google/Craigslist search forms** — use URL parameters instead
- **Not waiting after navigation** — pages need time to render before snapshot/interaction
- **Forgetting the surface reference** — capture it from `new-pane` output and reuse throughout

## Large Output Handling

When `snapshot` or `eval` results exceed the inline display limit, the output is auto-persisted to a file. The file path appears in the tool output — use `Read` to access the full data.
