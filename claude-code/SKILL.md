---
name: gemini
description: Use when the user asks to run Gemini CLI or references Google Gemini for code analysis, refactoring, or automated editing
user-invocable: true
---

# Gemini Skill Guide

## Overview

This skill enables Claude Code to invoke the Gemini CLI for code analysis, refactoring, and automated editing. It mirrors the codex skill's collaboration model but adapts to Gemini's CLI semantics.

## Available Models

| Model | Use case |
| --- | --- |
| `gemini-3-pro-preview` | Default. Most capable, best for complex tasks |
| `gemini-3-flash-preview` | Fast, good for simpler tasks and iteration |
| `gemini-2.5-pro` | Previous generation, stable |
| `gemini-2.5-flash` | Fast previous generation |
| `gemini-2.5-flash-lite` | Lightweight, lowest latency |

## Running a Task

### Quick Launch (preferred): `gm-run`

Use `gm-run` (`scripts/gm-run`) for the full lifecycle — launch, session registry, PID monitoring, state files, result JSON. Mirrors `cx-run` from the Codex skill.

```bash
$HOME/.claude/skills/gemini/scripts/gm-run --timeout 600 -- \
  gemini -m gemini-3-pro-preview -y -p "$PROMPT"
```

`gm-run` automatically:
- Injects `-o stream-json` to get structured JSONL with `session_id` and token/timing stats
- Launches gemini in background, monitors PID
- Extracts `session_id` from the stream's `init` event
- Registers in `.coord/sessions.jsonl` (with real `session_id`, not volatile index)
- Manages `.coord/running/done/failed/` state files with heartbeat
- Outputs result JSON on completion
- Handles timeout (kills process, writes failed state)

**Output** (JSON to stdout on completion):
```json
{"status":"completed","session_id":"uuid","response":"...","response_truncated":false,"total_tokens":18611,"output_tokens":31,"duration_ms":4274,"tool_calls":1,"elapsed_seconds":13}
```

**Options:**
- `--timeout SECONDS` — max wait (default: 600)
- `--max-chars N` — truncate response (default: 12288)
- `--coord-id ID` — custom coord ID (default: `gm_<timestamp>`)
- `--coord-dir DIR` — .coord directory (default: `.coord`)
- `--cwd DIR` — working directory (gemini has no `-C` flag, so gm-run `cd`s for you)
- `--resume INDEX` — resume a previous session by index or `"latest"`

Run with `run_in_background` from Claude Code — you'll be notified on completion.

### Launch Steps

1. **Guard check**: scan `.coord/running/` for active entries. If any exist for this project, do NOT proceed — wait for notification or resume.
2. Confirm the Task Package is complete.
3. Ask the user which model (default: `gemini-3-pro-preview`) and approval mode in a **single prompt**.
4. **For long/multiline prompts**, write to a temp file: `PROMPT=$(cat /path/to/task.txt)`.
5. **Launch with `gm-run`** using `run_in_background`:
   ```bash
   $HOME/.claude/skills/gemini/scripts/gm-run --timeout 600 -- \
     gemini -m gemini-3-pro-preview -y -p "$PROMPT"
   ```
   For worktree mode: add `--cwd /absolute/path/to/worktree`.
6. **Tell the user** Gemini is running, then **stop**. Do NOT poll or edit files.
7. **When notified**, read the result JSON and decide:
   - `completed` → proceed to Accept phase
   - `crashed` → investigate or resume
   - `timeout` → resume with `gm-run --resume <index> --timeout 600 -- -p "Continue"`

### Direct Command Patterns (without gm-run)

For simple one-off commands where lifecycle management isn't needed:

```bash
# Execute (headless)
gemini -m gemini-3-pro-preview --approval-mode auto_edit -p "<task package>" 2>/dev/null

# Read-only analysis
gemini -m gemini-3-pro-preview --approval-mode plan -p "<prompt>" 2>/dev/null

# Full auto (YOLO)
gemini -m gemini-3-pro-preview -y -p "<prompt>" 2>/dev/null

# With structured stream output (session_id + stats)
gemini -m gemini-3-pro-preview -y -o stream-json -p "<prompt>"
```

**IMPORTANT**: Append `2>/dev/null` to suppress noisy stderr on direct commands. **On non-zero exit**, re-run without `2>/dev/null` to capture error details.

## Approval Mode Selection

Choose the minimum required access. Prefer least privilege.

| Use case | Gemini flags | Description |
| --- | --- | --- |
| Read-only analysis | `--approval-mode plan` | Gemini plans but cannot write files. **Requires `experimental.plan` enabled in Gemini config.** |
| Auto-approve edits | `--approval-mode auto_edit` | Gemini can write files, auto-approves edits |
| Full auto (YOLO) | `-y` or `--approval-mode yolo` | No approval prompts at all |

**Note:** `--approval-mode plan` requires the experimental plan feature to be enabled. If unavailable, use `--approval-mode auto_edit` without `-y` as the least-privilege alternative for tasks that need file access.

If unsure, default to `--approval-mode auto_edit` and ask if full auto is needed.

## Working Directory Handling

Gemini has **no `-C` flag** for specifying a working directory. For parallel worktree mode, you must `cd` to the worktree directory first:

```bash
# Run Gemini in a worktree (subshell to preserve caller's cwd)
(cd ../<project>-gemini && gemini --approval-mode auto_edit -y -p "<prompt>" 2>/dev/null)
```

For serial mode in the current project directory, no special handling is needed.

## Session Management

### Recording Sessions

When using `gm-run`, sessions are registered automatically with the real `session_id` extracted from Gemini's stream-json `init` event. No manual registration needed.

For direct commands, record a session in `.coord/sessions.jsonl`:

```json
{"agent":"gemini","session_id":"<uuid-from-stream>","session_ref":{"type":"id","value":"<uuid>"},"task":"<short task description>","status":"running","cwd":"<working dir>","mode":"serial","created_at":"<ISO8601>"}
```

**Tip:** Use `-o stream-json` to capture the `session_id` from the `init` event. This is more stable than index-based references.

### Listing Sessions

```bash
gemini --list-sessions
```

This outputs all Gemini sessions. Match by `session_id` (if captured via stream-json) or by task description/timestamp to find the correct index for resume.

### Context Recovery

When Claude Code loses context:
1. Scan `.coord/running/` for active gemini state files — fastest way to detect in-flight tasks.
2. Read `.coord/sessions.jsonl` (`tail -n 50`) to find `agent: "gemini"` entries.
3. If a `done/` or `failed/` file exists for the coord_id, the task already finished — read the result.
4. Otherwise, run `gemini --list-sessions`, match by session_id or task description, and resume.

## Resume Protocol

When resuming a Gemini session, always use this protocol:

1. **Re-resolve the session index.** Gemini session indices are volatile and can shift as new sessions are created. Always run `gemini --list-sessions` first and match by task/timestamp to find the correct current index.

2. **Resume with a self-check prompt.** The first message to a resumed session must ask Gemini to summarize its current state:
   ```
   Resume. Before continuing, output a brief status summary:
   1. What has been completed so far
   2. What files were modified
   3. What remains to be done
   4. Any blockers or decisions needed
   Then continue with the remaining work.
   ```

3. **Resume command:**
   ```bash
   gemini --resume <resolved-index> -p "<resume prompt>" 2>/dev/null
   ```

4. **After timeout or error**: temporarily remove `2>/dev/null` to capture diagnostic output.

5. **Update the registry** after resume completes (status, updated_at, notes). Update `session_ref.value` with the newly resolved index.

## Task Package Format

Gemini uses the same standardized Task Package as Codex. The only addition is the `Agent` field:

```
[Task Package]
Agent: gemini
Goal:
- <single-sentence goal>

Mode: serial | parallel
Branch: <branch name, required for parallel mode>
Worktree: <worktree path, required for parallel mode>

Files:
- <explicit file list>

Ownership:
- <agent>: <file globs this agent may edit>
- DO NOT TOUCH: <file globs reserved for other agents>

Constraints:
- <non-functional requirements, style rules, time limits, deps>

Acceptance Criteria:
- <observable outcomes>
- <tests to run or checks to pass>

Notes:
- <assumptions, risk areas, or context snapshots>
```

For serial mode, `Mode`, `Branch`, `Worktree`, and `Ownership` fields may be omitted.

## Output Format

Gemini supports structured JSON output via the `-o json` flag. Use this when you need to parse results programmatically:

```bash
gemini -m gemini-3-pro-preview --approval-mode plan -o json -p "<prompt>" 2>/dev/null
```

When `-o json` is not used, Gemini outputs plain text which Claude Code should summarize for the user.

## Error Handling

- Stop and report failures whenever `gemini --version` or a `gemini` command exits non-zero; request direction before retrying.
- Before you use full-auto flags (`-y`, `--approval-mode yolo`) ask the user for permission using `AskUserQuestion` unless it was already given.
- When output includes warnings or partial results, summarize them and ask how to adjust using `AskUserQuestion`.
- **Parallel mode**: if worktree `cd` fails (directory doesn't exist), fall back to serial mode and inform the user.
- **Timeout recovery**: if a `gemini` command is interrupted, the session may still be recoverable. Look up the session from `.coord/sessions.jsonl`, re-resolve the index via `gemini --list-sessions`, then resume using the Resume Protocol.
- Unlike Codex, Gemini does **not** need `--skip-git-repo-check`.

## Acceptance Workflow (Claude Code)

After Gemini execution, Claude Code should:

1. Perform diff review of modified files.
2. Run tests or checks listed in `Acceptance Criteria`.
3. Verify constraints and confirm no files outside the `Ownership` / `Files` boundary were modified.
4. For parallel mode: check `.coord/events.jsonl` for blockers or handoff requests.
5. Decide one of:
   - **Accept** — proceed to next task or commit
   - **Revise** — re-enter Execute phase with feedback
   - **Abort** — revert changes

## Following Up

- If the task package is incomplete, request clarification **before** running Gemini.
- After every `gemini` command, update `.coord/sessions.jsonl` and use `AskUserQuestion` to confirm next steps.
- Restate the chosen model and approval mode when proposing follow-up actions.
- If conversation context has been lost, read `.coord/sessions.jsonl` (filtered to `agent: "gemini"`) to recover active session state before taking action.
