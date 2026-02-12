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

1. Confirm the Task Package exists and is complete.
2. Ask the user (via `AskUserQuestion`) which model to run (default: `gemini-3-pro-preview`) AND which approval mode to use (see Approval Mode Selection) in a **single prompt with two questions**.
3. Assemble the command with the appropriate options:
   - `-m, --model <MODEL>` (default: `gemini-3-pro-preview`)
   - `--approval-mode <plan|auto_edit|yolo>` or `-y` for full auto
   - `-p "<task package prompt>"` for headless execution
   - `-o json` for structured output (optional)
4. **IMPORTANT**: Append `2>/dev/null` to all `gemini` commands to suppress noisy stderr. Only show stderr if the user explicitly requests it or if debugging is needed. **On non-zero exit**, re-run without `2>/dev/null` to capture error details before reporting.
5. Run the command, capture output, and summarize the outcome for the user.
6. **After launching Gemini**, record a session entry in `.coord/sessions.jsonl` (see Session Management).
7. **After Gemini completes**, update the registry entry status to `done` and inform the user: "You can resume this Gemini session at any time by saying 'gemini resume' or asking me to continue."

### Command Patterns

```bash
# Execute (headless)
gemini -m gemini-3-pro-preview --approval-mode auto_edit -p "<task package>" 2>/dev/null

# Read-only analysis
gemini -m gemini-3-pro-preview --approval-mode plan -p "<prompt>" 2>/dev/null

# Full auto (YOLO)
gemini -m gemini-3-pro-preview -y -p "<prompt>" 2>/dev/null

# With structured output
gemini -m gemini-3-pro-preview --approval-mode auto_edit -o json -p "<prompt>" 2>/dev/null
```

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

After launching Gemini, record a session in `.coord/sessions.jsonl`:

```json
{"agent":"gemini","session_id":"gm-<timestamp>","session_ref":{"type":"index","value":0},"task":"<short task description>","status":"running","cwd":"<working dir>","mode":"serial","created_at":"<ISO8601>"}
```

The `session_ref.value` is the session index at time of creation. Since Gemini uses index-based session references that can shift, **always re-resolve the index before resuming**.

### Listing Sessions

```bash
gemini --list-sessions
```

This outputs all Gemini sessions. Parse the output to find the session matching your task description or timestamp.

### Context Recovery

When Claude Code loses context:
1. Read `.coord/sessions.jsonl` to find entries with `agent: "gemini"` and `status: "running"`.
2. Run `gemini --list-sessions` to get the current session list.
3. Match the recorded task description against the session list to find the correct current index.
4. Resume using the resolved index (see Resume Protocol).

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
