---
argument-hint: '[TICKET-123] [--all] [--deep] [--push] [--close <issue_numbers>]'
disable-model-invocation: false
effort: medium
name: commit
user-invocable: true
description: 'This skill should be used when the user asks to commit changes, craft a commit message, or run a commit workflow. Creates atomic git commits formatted as `TICKET-123: message` (ticket derived from the branch name), with optional deep analysis or push. Flags: --all, --deep, --close, --push.'
---

# Git Commit

Create atomic commits by staging the right files, analyzing the staged diff, composing a `TICKET-123: message` subject, and optionally pushing. Branches are named after the Jira ticket being worked on, so the ticket key is read from the current branch.

## Workflow

### 1) Pre-flight + context (single call)

Run all checks and context collection in one bash call:

```bash
git rev-parse --is-inside-work-tree \
  && ! test -d "$(git rev-parse --git-dir)/rebase-merge" \
  && ! test -f "$(git rev-parse --git-dir)/MERGE_HEAD" \
  && ! test -f "$(git rev-parse --git-dir)/CHERRY_PICK_HEAD" \
  && git symbolic-ref HEAD \
  && git status --short --branch
```

If any check fails, stop with a clear error and suggested fix.

Arguments: `$ARGUMENTS`

### 2) Parse arguments

- Flags:
  - `--all` commit all changes
  - `--deep` deep semantic analysis, concise WHY-focused body
  - `--push` push after commit
  - `--close <issue_numbers>` append `Closes #N` trailers for listed issues (comma/space-separated)
- Value arguments:
  - Ticket key matching `[A-Z][A-Z0-9]+-[0-9]+` (e.g. `PROJ-123`) overrides the branch-derived ticket
  - Quoted text overrides the inferred description

### 3) Stage + read diff

- If `--all`:
  - If no changes at all: error "No changes to commit"
  - If unstaged changes exist: `git add -A`
  - If already staged: proceed
- Otherwise (atomic commits):
  - Session-modified files = files edited in this session
  - Currently staged files: `git diff --cached --name-only`
  - For staged files NOT in session-modified set: `git restore --staged <file>`
  - For session-modified files with changes: `git add <file>`
  - If none: error "No files modified in this session"
- **Unrelated changes**: session-modified files may contain pre-existing uncommitted changes (hunks not from this session). Include the entire file—partial staging is impractical. Never revert, discard, or `git checkout` unrelated changes.
- Read the staged diff once: `git diff --cached`
- Log staged files with status (A/M/D)

### 4) Analyze + compose message

Read the staged diff and produce the commit message in a single pass.

**Ticket key** — resolve in order, first match wins:

1. Ticket argument (`[A-Z][A-Z0-9]+-[0-9]+`, e.g. `PROJ-123`) if provided.
2. First `[A-Z][A-Z0-9]+-[0-9]+` match in the current branch name.

If neither yields a ticket, stop and ask for one — never guess or commit without it.

**Unrelated hunks** — ignore pre-existing changes when determining the description. If unrelated changes share a file with session changes, they are included in the commit but must not influence the message.

**Message format:**

- Subject: `TICKET-123: description` (keep the whole line \<= 72 chars)
- Description in imperative mood ("add" not "added"), no trailing period
- Describe what the change does, not which files changed
- Body: hyphenated lines for distinct changes; skip for trivial changes

**Issue linking** — scan the chat transcript for GitHub issue references (e.g. `#123`, `owner/repo#123`, issue URLs) that the current changes resolve. For each match, append a `Closes #N` trailer. Skip issues merely mentioned in passing; include only ones the commit actually closes.

**If `--deep`:**

- Deep semantic analysis of the diff
- Body: 2-3 hyphenated lines max, focused on WHY

**If `--close`:**

- Append a `Closes #N` line for each issue number provided
- Multiple issues: one `Closes #N` per line in the body/trailer
- Merge with transcript-scanned issues; de-duplicate

### 5) Commit

- Use `git commit -m "subject"` (add `-m "body"` only if body is non-empty)
- Output: commit hash + subject + file count summary
- If failed: show error + suggest fix

### 6) Push (if `--push`)

- If upstream exists: `git push`
- If no upstream: `git push -u origin HEAD`
- If failed: show error + suggest fix (pull/rebase first, set upstream, check auth)
