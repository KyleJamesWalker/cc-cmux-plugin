# cmux Integration

This session runs inside cmux, a native macOS terminal for AI coding agents built on the Ghostty engine. Check `$CMUX_WORKSPACE_ID` to confirm you're inside cmux before using cmux commands.

<env_vars>
| Variable | Purpose |
|----------|---------|
| `CMUX_WORKSPACE_ID` | Current workspace reference |
| `CMUX_SURFACE_ID` | Current surface (pane/tab) reference |
| `CMUX_SOCKET_PATH` | Unix socket for API calls |
</env_vars>

<sidebar>
```bash
cmux set-status <key> <value> [--icon <name>] [--color <#hex>]
cmux clear-status <key>
cmux set-progress <0.0-1.0> [--label <text>]
cmux clear-progress
cmux log [--level <level>] [--source <name>] -- <message>
cmux notify --title <text> [--subtitle <text>] [--body <text>]
```
</sidebar>

<pane_management>
```bash
cmux new-split <left|right|up|down>
cmux new-pane [--type <terminal|browser>] [--direction <left|right|up|down>] [--url <url>]
cmux send [--surface <ref>] <text>
cmux send-key [--surface <ref>] <key>
cmux read-screen [--surface <ref>] [--scrollback] [--lines <n>]
cmux list-panes | list-workspaces | tree [--all] | identify [--workspace <ref>]
```
</pane_management>

<browser>
All browser commands follow: `cmux browser [--surface <ref>] <subcommand> [args]`

The `--surface` flag goes BEFORE the subcommand — placing it after will error. Capture the `surface=surface:N` from `open-split` to target subsequent commands. When omitted, `$CMUX_SURFACE_ID` is used.

Subcommands (omitting the `cmux browser` prefix):

```
open [url]                          open-split [url]
goto|navigate <url>                 back | forward | reload
url|get-url

snapshot [-i] [--cursor] [--compact] [--max-depth <n>] [--selector <css>]
screenshot [--out <path>] [--json]
get <url|title|text|html|value|attr|count|box|styles>
is <visible|enabled|checked> <selector>
console <list|clear>                errors <list|clear>
highlight <selector>

click|dblclick|hover|focus|check|uncheck|scroll-into-view <selector>
type <selector> <text>              fill <selector> [text]
press|keydown|keyup <key>           select <selector> <value>
scroll [--selector <css>] [--dx <n>] [--dy <n>]
eval <script>
wait [--selector <css>] [--text <text>] [--url-contains <text>] [--load-state <state>] [--function <js>] [--timeout-ms <ms>]
find <role|text|label|placeholder|alt|title|testid|first|last|nth> ...

frame <selector|main>               dialog <accept|dismiss> [text]
download [wait] [--path <path>]      upload_file <selector> <path>

identify [--surface <ref>]           tab <new|list|switch|close|<index>>
cookies <get|set|clear>              storage <local|session> <get|set|clear>
state <save|load> <path>
addinitscript <script>               addscript <script>
addstyle <css>
```

Most interaction/navigation subcommands accept `--snapshot-after` to return a snapshot after the action.

**Browser automation workflow — practical patterns:**

1. **Open a browser pane and capture the surface reference:**
   ```bash
   cmux new-pane --type browser --direction right --url "https://example.com"
   # => OK surface:35 pane:29 workspace:3
   # Always capture and reuse the surface:N reference for all subsequent commands.
   ```

2. **Wait for navigation, then snapshot:**
   After `navigate` or `goto`, the page needs time to load. Use a short delay (under 2s) then `snapshot --compact` to inspect the page structure:
   ```bash
   sleep 1.5 && cmux browser --surface surface:N snapshot --compact
   ```
   `--compact` keeps the output manageable. Use `--selector <css>` to scope to a specific area if the page is large.

3. **Prefer URL navigation over form interaction for search engines:**
   Google, Craigslist, and other search sites often have comboboxes/autocomplete widgets that throw JS exceptions when automated via `click`/`type`/`fill`. **Bypass forms entirely** by constructing the search URL directly:
   ```bash
   # WRONG — fragile, JS exceptions on complex widgets
   cmux browser --surface surface:N click "ref=e591"
   cmux browser --surface surface:N type "ref=e591" "search query"

   # RIGHT — reliable, skips form interaction entirely
   cmux browser --surface surface:N navigate "https://www.google.com/search?q=search+query"
   cmux browser --surface surface:N navigate "https://losangeles.craigslist.org/search/sss?query=dressers"
   ```
   This pattern works for most sites with query-string-based search.

4. **Use `eval` for JavaScript execution** (not `js`, `evaluate`, or `exec`):
   ```bash
   cmux browser --surface surface:N eval "document.title"
   cmux browser --surface surface:N eval "JSON.stringify(someData, null, 2)"
   ```

5. **Extract structured data with `eval`:**
   Use `eval` with `JSON.stringify` and CSS selectors to scrape page data into JSON. Pipe directly to a file:
   ```bash
   cmux browser --surface surface:N eval "JSON.stringify(
     Array.from(document.querySelectorAll('a.titlestring')).map(a => ({
       title: a.textContent.trim(),
       link: a.href
     })), null, 2)" > results.json
   ```

6. **Use `screenshot --out <path>` for visual capture:**
   ```bash
   cmux browser --surface surface:N screenshot --out /path/to/output.jpg
   ```
   Screenshots capture the current viewport. Scroll first if needed:
   ```bash
   cmux browser --surface surface:N scroll --dy 500
   cmux browser --surface surface:N screenshot --out /path/to/below-fold.jpg
   ```

7. **Use `get` for simple value extraction without JavaScript:**
   ```bash
   cmux browser --surface surface:N get url      # current URL
   cmux browser --surface surface:N get title    # page title
   cmux browser --surface surface:N get text --selector "h1"  # element text
   cmux browser --surface surface:N get html --selector ".results"  # element HTML
   cmux browser --surface surface:N get count --selector "li.result"  # count matches
   ```

8. **Interaction fallback order** when a direct approach fails:
   - First try: `navigate` with URL parameters (most reliable)
   - Second try: `fill` on the selector (sets value programmatically)
   - Third try: `click` then `type` (simulates real user input)
   - Last resort: `eval` with raw JavaScript DOM manipulation

9. **Large outputs get auto-persisted** — when a snapshot or eval result exceeds the inline limit, it's saved to a file. Read the persisted file path from the output to access the full data.
</browser>

<teams_and_agents>
When spawning teams or agents, make them visible in cmux panes instead of running as invisible background subprocesses.

**CRITICAL: `cmux send` does NOT press Enter.** You must always follow `cmux send` with a separate `cmux send-key "Return"` to execute the command. This is the #1 source of stuck agents.

```bash
# CORRECT - two-step: type, then press Enter
cmux send --surface surface:N "some command"
cmux send-key --surface surface:N "Return"

# WRONG - command is typed but never executed
cmux send --surface surface:N "some command"
```

**Workflow — Interactive Claude Sessions (preferred):**
1. Name the lead/coordinator session first: `/rename commander` (or similar) so it's identifiable in `cmux tree`.
2. Create split panes — use `cmux new-split right` and `cmux new-split down` for a multi-pane layout (lead on left, agents on right stacked vertically).
3. Capture the `surface:N` reference returned by `cmux new-split`.
4. Launch claude with a descriptive `--name` in each pane — **do NOT `cd` first** (see Git Worktrees note below):
   ```bash
   cmux send --surface surface:N "claude --dangerously-skip-permissions --name agent-name-here"
   cmux send-key --surface surface:N "Return"
   ```
   Use names matching the agent's role (e.g., `ui-ux-engineer`, `logic-dev`, `architect`).
5. **Wait for the `❯` prompt** before sending the task — poll instead of using a fixed sleep:
   ```bash
   # Condition-based init wait — poll every 0.5s until the claude prompt appears
   for i in $(seq 1 40); do
     output=$(cmux read-screen --surface surface:N --lines 3 2>&1)
     echo "$output" | grep -q "❯" && break
     sleep 0.5
   done
   ```
   Then send the task. For long/complex prompts write to a uniquely-named file and point the agent at it (avoids quoting issues and collisions between agents):
   ```bash
   # Write prompt to /tmp/cmux-task-AGENTNAME.txt with the Write tool, then:
   cmux send --surface surface:N "Read /tmp/cmux-task-AGENTNAME.txt and follow the instructions in it"
   cmux send-key --surface surface:N "Return"
   ```
   **Always include sentinel instructions at the end of every task prompt** (see Sentinel Protocol below).
6. **Monitor via sentinel files** — poll `/tmp/cmux-done-AGENTNAME.json` for completion rather than parsing terminal output. Use `cmux read-screen --surface surface:N --scrollback --lines <n>` only for debugging a specific agent.
7. Check file changes to confirm agent output (e.g., `wc -l`, `Read`).
8. Use `cmux tree --all` to verify all agents are running and named correctly.

**Full example — launching a named agent with sentinel completion:**
```bash
# 1. Create pane
cmux new-split right
# => OK surface:7 workspace:2

# 2. Launch claude with a name (from current dir — do NOT cd first)
cmux send --surface surface:7 "claude --dangerously-skip-permissions --name ui-ux-engineer"
cmux send-key --surface surface:7 "Return"

# 3. Wait for ❯ prompt (condition-based, not fixed sleep)
for i in $(seq 1 40); do
  cmux read-screen --surface surface:7 --lines 3 | grep -q "❯" && break
  sleep 0.5
done

# 4. Send the task (prompt file must include sentinel instructions)
cmux send --surface surface:7 "Read /tmp/cmux-task-ui-ux-engineer.txt and follow the instructions in it"
cmux send-key --surface surface:7 "Return"

# 5. Poll for completion (no screen parsing needed)
while [ ! -f /tmp/cmux-done-ui-ux-engineer.json ]; do sleep 5; done
cat /tmp/cmux-done-ui-ux-engineer.json
```

**Git Worktrees — avoid creating extra CC sessions/projects:**
When you `cd /path/to/worktree && claude`, Claude Code registers the worktree directory as a **new project**, cluttering the projects list. Avoid this by:

- **Preferred:** Launch `claude` without `cd`-ing — it stays in the current (main) project dir. Pass the worktree path as an absolute path in the task prompt so the agent uses it for all edits and commands:
  ```bash
  # WRONG — creates a new CC project entry for the worktree
  cmux send --surface surface:N "cd /path/duck_data_wt_main && claude --name agent"
  cmux send-key --surface surface:N "Return"

  # RIGHT — stays in main project, agent uses absolute worktree path in task
  cmux send --surface surface:N "claude --dangerously-skip-permissions --name agent"
  cmux send-key --surface surface:N "Return"
  # In the task prompt: "Work only in /path/duck_data_wt_main — use absolute paths for all edits"
  ```
- **Alternative:** Create worktrees under `/tmp/` so they never appear as persistent project entries:
  ```bash
  git worktree add /tmp/wt-feature HEAD
  ```

**Pane reuse — CWD persists between tasks:**
When reassigning an idle pane to a new task, the agent's shell state (including cwd) is unchanged from the previous task. Always use absolute paths in task prompts and explicitly state the working directory — don't assume the agent is in the directory you expect.

**Agent completion — sentinel file protocol:**

This is the preferred way for sub-agents to report back. It replaces `cmux send` callbacks (which have race conditions) and `cmux read-screen` polling (which requires fragile terminal output parsing).

*What to append to every sub-agent task prompt:*
```
When your task is fully complete:
1. Write your full results to /tmp/cmux-results-AGENTNAME.md
2. Write a JSON sentinel atomically — results file MUST be written first:

   printf '{"agent":"AGENTNAME","status":"success","summary":"one-line summary here","results":"/tmp/cmux-results-AGENTNAME.md"}' \
     > /tmp/cmux-done-AGENTNAME.tmp.json \
     && mv /tmp/cmux-done-AGENTNAME.tmp.json /tmp/cmux-done-AGENTNAME.json

   On failure, use "status":"error" and put the error message in "summary".
   The mv makes the write atomic — the coordinator only sees a complete sentinel.
```

*Sentinel JSON structure:*
```json
{
  "agent": "ui-ux-engineer",
  "status": "success",
  "summary": "Built login form with validation, 3 components added",
  "results": "/tmp/cmux-results-ui-ux-engineer.md"
}
```

*Main agent polling — wait for all sentinels:*
```bash
agents=("ui-ux-engineer" "logic-dev" "architect")
timeout=1800  # 30 min max
elapsed=0
interval=10

while [ $elapsed -lt $timeout ]; do
  all_done=true
  for agent in "${agents[@]}"; do
    [ -f "/tmp/cmux-done-${agent}.json" ] || { all_done=false; break; }
  done
  $all_done && break
  sleep $interval
  elapsed=$((elapsed + interval))
done

# Read results
for agent in "${agents[@]}"; do
  sentinel="/tmp/cmux-done-${agent}.json"
  if [ -f "$sentinel" ]; then
    cat "$sentinel"   # shows status + summary
  else
    echo "TIMEOUT/CRASH: ${agent} never completed"
  fi
done
```

Why this beats polling `cmux read-screen`:
- **No terminal output parsing** — file existence is a single syscall
- **Atomic** — coordinator never reads a partial sentinel (`mv` is atomic on the same filesystem)
- **Crash detection** — a missing sentinel after timeout means the agent died or hung
- **Results on disk** — coordinator reads the full results file only when needed, not streamed through terminal state
- **No race conditions** — each agent writes its own file; simultaneous completions don't interfere

*Cleanup sentinels after all agents are done:*
```bash
rm -f /tmp/cmux-done-*.json /tmp/cmux-results-*.md /tmp/cmux-task-*.txt
```

**Stuck agents — interrupt extended thinking:**
A missing sentinel after the polling timeout is the first signal an agent may be stuck or crashed. Verify with `cmux read-screen` before intervening:
```bash
cmux read-screen --surface surface:N --scrollback --lines 20
```
If it shows the same "Thinking..." / "Ionizing..." state for more than 3-4 minutes, send Escape and retry with a simpler prompt:
```bash
cmux send-key --surface surface:N "Escape"
sleep 2
cmux send --surface surface:N "your simplified task here"
cmux send-key --surface surface:N "Return"
```
If the agent has crashed entirely (no activity, no sentinel), close the pane and re-spawn:
```bash
cmux close-surface --surface surface:N
# Re-run the spawn + task steps for that agent
```

**Why interactive mode over `claude -p`:** Using `claude -p "..."` requires the entire prompt on the command line, which causes quoting/escaping nightmares when sent through `cmux send` (nested quotes, special characters, multi-line content all break). Interactive mode avoids this entirely — claude is already running, and `cmux send` just types the message into the session.

**Fallback — Agent tool:** If cmux pane agents aren't working (e.g., socket access issues, sandbox restrictions), fall back to the `Agent` tool for parallel work. Agents write to separate files so there are no conflicts. Use cmux `set-progress`/`set-status`/`notify` to keep the user informed of progress even when using invisible agents.

**Do NOT** spawn agents via the `Agent` tool with `team_name` when running inside cmux — they run as invisible background processes with no terminal visibility. Prefer cmux panes + `cmux send`/`cmux send-key` so the user can watch each agent work in real time.

**Layout patterns:**
- 2 agents: left (lead) + right (agent), or top (lead) + bottom (agent)
- 3 agents: left (lead) + top-right (agent 1) + bottom-right (agent 2)
- Browser preview: add a `--type browser --direction down` pane for live preview

**Progress tracking:**
- Use `cmux set-progress` to show overall team progress in the status bar.
- Use `cmux set-status` to show which agents are active and their current phase.
- Use `cmux notify` when an agent completes its work or hits a blocker.

**Cleanup:**
- Agents launched via `claude -p` exit automatically when done.
- Interactive agents need `/exit` before closing their pane — send it first, then close:
  ```bash
  cmux send --surface surface:N "/exit"
  cmux send-key --surface surface:N "Return"
  sleep 1
  cmux close-surface --surface surface:N
  ```
- If the agent is unresponsive, fall back to Ctrl+C: `cmux send-key --surface surface:N "ctrl+c"`.
</teams_and_agents>

<guidelines>
- Prefer `open-split` for web previews; prefer `right` or `down` splits.
- Prefer `snapshot` over `screenshot` for programmatic inspection (returns accessibility tree).
- Use `set-progress` for long multi-step tasks; `log` for milestones; `notify` for alerts needing attention across panes.
- Use `--snapshot-after` to chain inspection with interaction in a single call.
- When using teams/agents, always spawn them in visible cmux panes so the user can observe their work.
</guidelines>
