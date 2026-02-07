# Gemini Integration for Claude Code

## Purpose

Enable Claude Code to invoke the Gemini CLI for automated code analysis, refactoring, and editing workflows. Part of the multi-agent skill architecture alongside the codex and multi-agent skills.

## Prerequisites

- `gemini` CLI installed and available on `PATH`.
- Gemini configured with valid Google credentials.
- Confirm the installation by running `gemini --version`; resolve any errors before using the skill.

## Installation

Download this repo and store the skill in `~/.claude/skills/gemini`:

```bash
git clone --depth 1 <repo-url> /tmp/skill-gemini-temp && \
mkdir -p ~/.claude/skills && \
cp -r /tmp/skill-gemini-temp/ ~/.claude/skills/gemini && \
rm -rf /tmp/skill-gemini-temp
```

Or, if using the `my-skills` installer:

```bash
# Copy into my-skills directory structure
cp -r skill-gemini my-skills/skills/gemini/claude-code/
cd my-skills && ./install.sh gemini
```

## Usage

### Example Workflow

**User prompt:**
```
Use gemini to analyze this repository for potential refactoring opportunities.
```

**Claude Code response:**
Claude will activate the Gemini skill and:
1. Ask which model to use (e.g., `gemini-2.5-pro`).
2. Ask which approval mode (`plan`, `auto_edit`, or `yolo`).
3. Run a command like:
```bash
gemini -m gemini-2.5-pro --approval-mode plan \
  -p "Analyze this repository for refactoring opportunities..." 2>/dev/null
```

### Approval Modes

| Mode | Flag | Use case |
| --- | --- | --- |
| Plan (read-only) | `--approval-mode plan` | Analysis, code review |
| Auto-edit | `--approval-mode auto_edit` | Automated refactoring |
| YOLO (full auto) | `-y` | Trusted batch operations |

### Session Management

Gemini sessions are tracked in `.coord/sessions.jsonl` alongside Codex sessions, using the `agent: "gemini"` field. Sessions can be resumed via index-based references.

**Important**: Gemini session indices are volatile. Always re-resolve via `gemini --list-sessions` before resuming.

### Resume

```bash
gemini --list-sessions                              # Find correct index
gemini --resume <index> -p "<resume prompt>" 2>/dev/null
```

### Working Directory

Gemini has no `-C` flag. For parallel worktree mode, use a subshell:
```bash
(cd ../<project>-gemini && gemini --approval-mode auto_edit -y -p "<prompt>" 2>/dev/null)
```

### Detailed Instructions

See `SKILL.md` for complete operational instructions, CLI options, and workflow guidance.
