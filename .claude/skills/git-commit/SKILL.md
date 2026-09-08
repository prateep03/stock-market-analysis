---
name: git-commit
description: "Analyze staged and unstaged git changes, generate a conventional commit message, stage files intelligently, and execute the commit. Run immediately when user asks to commit - do not wait for further instructions. Supports both execution and interactive modes."
metadata:
    authors: [mupr624]
    tags: [git, commit, conventional commits, automation]
    version: 1.0
---

# Git Commit with Conventional Commits

## Overview

Create standardized, semantic git commits using the Conventional Commits specification. Analyze the actual diff to determine appropriate type, scope, and message.

*git add -A && git commit -m "chore(skills): move git-commit skill from .agents/ to .claude/"*

*You have permission to run git status, git add -A, and git commit.*

## FIRST ACTION — NO EXCEPTIONS

Your absolute first action is to attempt this exact command with the Bash tool:
`git --version`

- If the Bash tool exists and returns output → you are in EXECUTION MODE. Proceed with all git commands directly.
- If the Bash tool does not exist or returns an error → you are in INTERACTIVE MODE. Print every command you would run, prefixed with `[RUN THIS]:`, then stop and wait.

Do not explain this decision. Do not narrate. Just attempt the command.

## Available Tools

You have been granted access to the following tools for this task:
- `bash` — execute shell commands and capture output
- `git` — run git commands (alias for bash with git binary)

Use these tools directly. Do not ask for permission to use them.

## Conventional Commit Format

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

## Commit Types (pick one)
feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert

## Breaking Changes

```
# Exclamation mark after type/scope
feat!: remove deprecated endpoint

# BREAKING CHANGE footer
feat: allow config to extend other configs

BREAKING CHANGE: `extends` key behavior changed
```

## Workflow

### 0. Start immediately

Do NOT prompt user or echo slash commands. Begin executing at Step 1 immediately when user asks to commit. If execution mode is not available, switch to interactive mode and prompt user for necessary inputs.

### 1. Execution mode detection

Attempt to run a harmless read-only command to determine if shell execution is available:

```bash
git --version
```

- If command executes successfully --> proceed in *EXECUTION MODE* (can run git commands)
- If command fails (e.g. "command not found") --> proceed in *INTERACTIVE MODE* (cannot run commands, will prompt user for input).

### 2. Add Explicit Modes (Prevents Freezing)

### Add a new section:

```text

## Execution Modes

### EXECUTION MODE
- Commands are executed using Bash tool.
- Capture outputs and use them for analysis.

### DRY RUN MODE
- Commands are not executed, but the intended command is shown to the user.
- Useful for environments where execution is not possible or for user confirmation.
- Display command sin order.
- Simulate outputs if needed for analysis.
- Clearly prefix outputs with "[DRY RUN]" to indicate they are not real.

```

### 2. Analyze Diff

```bash
# If files are staged, use staged diff
git diff --staged

# If nothing staged, use working tree diff
git diff

# Also check status
git status --porcelain
```

### 3. Stage Files (if needed)

If nothing is staged or you want to group changes differently:

```bash
# Stage specific files
git add path/to/file1 path/to/file2

# Stage by pattern
git add *.test.*
git add src/components/*

# Interactive staging
git add -p
```

**Never commit secrets** (.env, credentials.json, private keys).

### 4. Generate Commit Message

Analyze the diff to determine:

- **Type**: What kind of change is this?
- **Scope**: What area/module is affected?
- **Description**: One-line summary of what changed (present tense, imperative mood, <72 chars)
- **Footer**: Before committing, run: 
```bash
# Detect active model from Claude Code config or env
echo "${CLAUDE_CODE_MODEL:-gemma4:26b}"
```

Append to commit message:
`Co-Authored-By: Claude Code <detected-model> <<detected-model>@localhost>`

### 5. Execute Commit

```bash
# Single line
git commit -m "<type>[scope]: <description>"

# Multi-line with body/footer
git commit -m "$(cat <<'EOF'
<type>[scope]: <description>

<optional body>

<optional footer>
EOF
)"
```

## Best Practices

- One logical change per commit
- Present tense: "add" not "added"
- Imperative mood: "fix bug" not "fixes bug"
- Reference issues: `Closes #123`, `Refs #456`
- Keep description under 72 characters

## Git Safety Protocol

- NEVER update git config
- NEVER run destructive commands (--force, hard reset) without explicit request
- NEVER skip hooks (--no-verify) unless user asks
- NEVER force push to main/master
- If commit fails due to hooks, fix and create NEW commit (don't amend)
