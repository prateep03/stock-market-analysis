---
name: git-push
description: "Push committed local changes to the remote git repository. Detects authentication method (SSH or HTTPS) and executes the appropriate push command. Run immediately when user asks to push - do not wait for further instructions." 
metadata:
    authors: [mupr624]
    tags: [git, push, remote, automation]
    version: 1.0
---

# Git Push

## FIRST ACTION — NO EXCEPTIONS

Attempt this command immediately with the Bash tool:
`git --version`

- Returns output → **EXECUTION MODE**: run all commands directly.
- Fails → **INTERACTIVE MODE**: print each command prefixed with `[RUN THIS]:` and stop.

Do not narrate. Do not explain. Just attempt the command.

## Available Tools
- `bash` — execute shell commands
- `git` — git binary via bash

## Workflow

### 1. Pre-push checks

```bash
git status --porcelain          # confirm nothing uncommitted that was intended
git log @{u}..HEAD --oneline    # list commits to be pushed (requires upstream set)
git remote -v                   # confirm remote URL and auth method
```

If `git log @{u}..HEAD` fails (no upstream set), run:
```bash
git branch --show-current       # get current branch name
```
Then push with explicit upstream: `git push -u origin <branch>`.

### 2. Authentication — detect and apply

Inspect the remote URL from `git remote -v` output:

| Remote URL pattern | Auth method | Notes |
|---|---|---|
| `git@github-partner.azc.ext.hp.com:...` | **SSH key** | Uses `~/.ssh/id_ed25519` or `id_rsa`. Verify: `ssh -T git@github-partner.azc.ext.hp.com` |
| `https://github-partner.azc.ext.hp.com/...` | **HTTPS + credential store** | Uses git credential helper or token in URL |
| `https://<token>@github-partner.azc.ext.hp.com/...` | **PAT embedded in URL** | Token baked into remote URL |
| `ssh://user@host:port/...` | **SSH to custom host** | e.g. DVC remote pattern `ssh://mupr624@15.77.13.60:22/...` |

**SSH troubleshooting** (if push fails with auth error):
```bash
ssh -T git@github-partner.azc.ext.hp.com           # test GitHub SSH auth
ssh-add -l                      # list loaded keys; if empty, run ssh-add ~/.ssh/id_ed25519
```

**HTTPS troubleshooting** (if push fails with 403/auth error):
```bash
git credential reject           # clear cached bad credentials
# Then retry push — credential helper will prompt for token
```

**Never** embed credentials in commands shown to the user. If a token is needed, instruct the user to set it via:
```bash
git remote set-url origin https://<YOUR_TOKEN>@github-partner.azc.ext.hp.com/<user>/<repo>.git
```

### 3. Execute push

```bash
# Standard push (upstream already set)
git push

# First push on new branch (sets upstream)
git push -u origin <branch>

# Push specific branch explicitly
git push origin <branch>:<branch>
```

### 4. Verify

```bash
git log @{u}..HEAD --oneline    # should return empty (all commits pushed)
```

If output is empty → push succeeded. Report the branch name and number of commits pushed.

## Safety Protocol

- NEVER force push (`--force`, `-f`) unless user explicitly requests it
- NEVER push to `main`/`master` directly if a PR workflow is in place — warn the user
- NEVER push tags automatically (`--follow-tags`) unless asked
- If push is rejected due to diverged history, report the conflict and suggest `git pull --rebase` — do not resolve automatically
- NEVER modify remote URLs or git config without explicit user instruction

## INTERACTIVE MODE output format

If execution is unavailable, output exactly:

```
[RUN THIS]: git remote -v
[RUN THIS]: git log @{u}..HEAD --oneline
[RUN THIS]: git push
```

Infer branch and remote from any context the user has provided.