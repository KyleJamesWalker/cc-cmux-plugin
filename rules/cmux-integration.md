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
cmux set-status <key> <value> [--icon <name>] [--color <#hex>] [--priority <n>]
cmux clear-status <key>
cmux list-status
cmux set-progress <0.0-1.0> [--label <text>]
cmux clear-progress
cmux log [--level <level>] [--source <name>] [--] <message>
cmux list-log [--limit <n>] | clear-log
cmux notify [--title <text>] [--subtitle <text>] [--body <text>] [--reply] [--clear]
cmux trigger-flash [--surface <ref>]
```
</sidebar>

<pane_management>
```bash
cmux new-split <left|right|up|down> [--command <text>] [--focus <true|false>] [--workspace <ref>] [--surface <ref>]
cmux new-pane [--type <terminal|browser|simulator>] [--direction <left|right|up|down>] [--placement <workspace|dock>]
              [--url <url>] [--profile <name>] [--command <text>] [--focus <true|false>] [--workspace <ref>]
cmux send [--surface <ref>] [--] <text>          # \n or \r sends Enter, \t sends Tab
cmux send-key [--surface <ref>] <key>
cmux read-screen [--surface <ref>] [--scrollback] [--lines <n>] [--selection]
cmux close-surface [--surface <ref>]
cmux rename-tab [--surface <ref>] <title>
cmux wait-for [-S|--signal] <name> [--timeout <seconds>]
cmux list-panes | list-pane-surfaces | list-workspaces
cmux tree [--all] [--json] | identify [--surface <ref>]
cmux markdown open <path> | diff [--staged|--branch|--last-turn] | todo <add|list|check|...>
```
</pane_management>

<browser>
All browser commands follow: `cmux browser [--surface <ref> | <ref>] <subcommand> [args]`

Every subcommand except `open`, `open-split`, `new` and `identify` needs a surface, passed as `--surface surface:N` or as the first word (`cmux browser surface:N get url`). `$CMUX_SURFACE_ID` is not a fallback: it points at your terminal. Capture the `surface=surface:N` from `open-split`, or find an existing browser surface with `cmux --json tree --all`.

Subcommands (omitting the `cmux browser` prefix):

```
open|open-split|new [url] [--focus <true|false>] [--profile <name>]
goto|navigate <url>                 back | forward | reload
get url                             focus-webview | is-webview-focused

snapshot [--interactive] [--cursor] [--compact] [--max-depth <n>] [--selector <css>]
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
wait [--selector <css>] [--text <text>] [--url-contains <text>] [--load-state <interactive|complete>] [--function <js>] [--timeout-ms <ms>|--timeout <s>]
find <role|text|label|placeholder|alt|title|testid|first|last|nth> ...

frame <selector|main>               dialog <accept|dismiss> [text]
download [wait] [--path <path>] [--timeout-ms <ms>]   download list [--limit <1-25>]
viewport <width> <height> | reset

identify [--surface <ref>]           tab <new|list|switch|close|<index>>
cookies <get|set|clear> [--url|--domain|--all ...]   storage <local|session> <get|set|clear>
profiles <list|add|rename|clear|delete>               import [--from <browser>]
state <save|load> <path>
addinitscript <script>               addscript <script>
addstyle <css>
```

Most interaction/navigation subcommands accept `--snapshot-after` to return a snapshot after the action.

For step-by-step browser automation workflows (searching, scraping, screenshots, form interaction), use the `cmux-integration:browser-automation` skill.
</browser>

<teams_and_agents>
When spawning teams or agents, make them visible in cmux panes instead of running as invisible background subprocesses.

**`cmux send` presses Enter only for a trailing `\n`.** Without it the text is typed but never run, which is the most common cause of stuck agents. To start a command in a new pane, pass `--command` to `new-split` or `new-pane` instead of a separate `send`.

```bash
# Start a command in a new pane: typed and run when the shell spawns
cmux new-split right --command "some command"

# Run a command in an existing pane
cmux send --surface surface:N "some command\n"

# WRONG - typed but never executed
cmux send --surface surface:N "some command"
```

**Workflow — Interactive Claude Sessions (preferred):**
1. Name the lead/coordinator session first: `/rename commander` (or similar) so it's identifiable in `cmux tree`.
2. Create split panes — use `cmux new-split right` and `cmux new-split down` for a multi-pane layout (lead on left, agents on right stacked vertically).
3. Launch claude with a descriptive `--name` through `--command` — **do NOT `cd` first** (see Git Worktrees note below):
   ```bash
   cmux new-split right --command "claude --dangerously-skip-permissions --name agent-name-here"
   ```
   Use names matching the agent's role (e.g., `ui-ux-engineer`, `logic-dev`, `architect`).
4. Capture the `surface:N` reference that `cmux new-split` prints.
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
   cmux send --surface surface:N "Read /tmp/cmux-task-AGENTNAME.txt and follow the instructions in it\n"
   ```
   **Always include sentinel instructions at the end of every task prompt** (see Sentinel Protocol below).
6. **Monitor via sentinel files** — poll `/tmp/cmux-done-AGENTNAME.json` for completion rather than parsing terminal output. Use `cmux read-screen --surface surface:N --scrollback --lines <n>` only for debugging a specific agent.
7. Check file changes to confirm agent output (e.g., `wc -l`, `Read`).
8. Use `cmux tree --all` to verify all agents are running and named correctly.

**Full example — launching a named agent with sentinel completion:**
```bash
# 1. Create pane and launch claude with a name (from current dir — do NOT cd first)
cmux new-split right --command "claude --dangerously-skip-permissions --name ui-ux-engineer"
# => OK surface:7 workspace:2

# 2. Wait for ❯ prompt (condition-based, not fixed sleep)
for i in $(seq 1 40); do
  cmux read-screen --surface surface:7 --lines 3 | grep -q "❯" && break
  sleep 0.5
done

# 3. Send the task (prompt file must include sentinel instructions)
cmux send --surface surface:7 "Read /tmp/cmux-task-ui-ux-engineer.txt and follow the instructions in it\n"

# 4. Block until the agent signals, then read its sentinel
cmux wait-for done-ui-ux-engineer --timeout 1800
cat /tmp/cmux-done-ui-ux-engineer.json
```

**Git Worktrees — avoid creating extra CC sessions/projects:**
When you `cd /path/to/worktree && claude`, Claude Code registers the worktree directory as a **new project**, cluttering the projects list. Avoid this by:

- **Preferred:** Launch `claude` without `cd`-ing — it stays in the current (main) project dir. Pass the worktree path as an absolute path in the task prompt so the agent uses it for all edits and commands:
  ```bash
  # WRONG — creates a new CC project entry for the worktree
  cmux new-split right --command "cd /path/duck_data_wt_main && claude --name agent"

  # RIGHT — stays in main project, agent uses absolute worktree path in task
  cmux new-split right --command "claude --dangerously-skip-permissions --name agent"
  # In the task prompt: "Work only in /path/duck_data_wt_main — use absolute paths for all edits"
  ```
- **Alternative:** Create worktrees under `/tmp/` so they never appear as persistent project entries:
  ```bash
  git worktree add /tmp/wt-feature HEAD
  ```

**Pane reuse — CWD persists between tasks:**
When reassigning an idle pane to a new task, the agent's shell state (including cwd) is unchanged from the previous task. Always use absolute paths in task prompts and explicitly state the working directory — don't assume the agent is in the directory you expect.

**Agent completion — sentinel file protocol:**

This is the preferred way for sub-agents to report back. It replaces `cmux send` callbacks (which have race conditions) and `cmux read-screen` polling (which requires fragile terminal output parsing). The results stay on disk, and `cmux wait-for` wakes the coordinator without polling.

*What to append to every sub-agent task prompt:*
```
When your task is fully complete:
1. Write your full results to /tmp/cmux-results-AGENTNAME.md
2. Write a JSON sentinel atomically — results file MUST be written first:

   printf '{"agent":"AGENTNAME","status":"success","summary":"one-line summary here","results":"/tmp/cmux-results-AGENTNAME.md"}' \
     > /tmp/cmux-done-AGENTNAME.tmp.json \
     && mv /tmp/cmux-done-AGENTNAME.tmp.json /tmp/cmux-done-AGENTNAME.json
3. Signal the coordinator: cmux wait-for -S done-AGENTNAME

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

*Main agent — wait for all sentinels:*
```bash
agents=("ui-ux-engineer" "logic-dev" "architect")

# A signal sent before the wait starts is kept, so the order agents finish in does not matter
for agent in "${agents[@]}"; do
  cmux wait-for "done-${agent}" --timeout 1800
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
- **No terminal output parsing** — `wait-for` returns on the signal instead of a sleep loop
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
cmux send --surface surface:N "your simplified task here\n"
```
If the agent has crashed entirely (no activity, no sentinel), close the pane and re-spawn:
```bash
cmux close-surface --surface surface:N
# Re-run the spawn + task steps for that agent
```

**Why interactive mode over `claude -p`:** Using `claude -p "..."` requires the entire prompt on the command line, which causes quoting/escaping nightmares when sent through `cmux send` (nested quotes, special characters, multi-line content all break). Interactive mode avoids this entirely — claude is already running, and `cmux send` just types the message into the session.

**Fallback — Agent tool:** If cmux pane agents aren't working (e.g., socket access issues, sandbox restrictions), fall back to the `Agent` tool for parallel work. Agents write to separate files so there are no conflicts. Use cmux `set-progress`/`set-status`/`notify` to keep the user informed of progress even when using invisible agents.

**Agent teams:** inside a plain `claude` session, the `Agent` tool with `team_name` runs teammates as invisible background processes. Prefer cmux panes so the user can watch each agent work. A session started with `cmux claude-teams [claude-args...]` opens its teammates as visible cmux splits instead.

**Layout patterns:**
- 2 agents: left (lead) + right (agent), or top (lead) + bottom (agent)
- 3 agents: left (lead) + top-right (agent 1) + bottom-right (agent 2)
- Browser preview: `cmux browser open-split <url>` for live preview

**Progress tracking:**
- Use `cmux set-progress` to show overall team progress in the status bar.
- Use `cmux set-status` to show which agents are active and their current phase.
- Use `cmux notify` when an agent completes its work or hits a blocker.

**Cleanup:**
- Agents launched via `claude -p` exit automatically when done.
- Interactive agents need `/exit` before closing their pane — send it first, then close:
  ```bash
  cmux send --surface surface:N "/exit\n"
  sleep 1
  cmux close-surface --surface surface:N
  ```
- If the agent is unresponsive, fall back to Ctrl+C: `cmux send-key --surface surface:N "ctrl+c"`.
</teams_and_agents>

<guidelines>
- Prefer `open-split` for web previews; prefer `right` or `down` splits.
- Prefer `snapshot` over `screenshot` for programmatic inspection (returns accessibility tree).
- Use `wait --load-state complete` after navigation instead of a fixed sleep.
- Use `set-progress` for long multi-step tasks; `log` for milestones; `notify` for alerts needing attention across panes.
- Use `--snapshot-after` to chain inspection with interaction in a single call.
- When using teams/agents, always spawn them in visible cmux panes so the user can observe their work.
</guidelines>
