---
name: handover
description: Save a rich handover document of the current session so a future Claude session can resume the work after /clear. Use when the user types /handover [optional-label] or asks to "save context", "checkpoint this session", "write a handover", or "I'm about to clear". Also invoked automatically by the PreCompact hook before context compaction. Pair with /handover-resume in the next session.
---

# Handover

Save the current session's state to a rich markdown doc so a future session can pick up the work with full context.

## Storage

Global, organized by project. Never lives inside the repo:

```
~/.claude/handovers/<project-slug>/<YYYY-MM-DD-HHMM>-<slug>.md
```

Compute the project slug from the **main** worktree's basename so all
worktrees of the same repo share one handover folder. Fall back to cwd if
not in a git repo.

```bash
project_root=$(git worktree list --porcelain 2>/dev/null | awk '/^worktree / {print $2; exit}')
[[ -z "$project_root" ]] && project_root=$(pwd)
project_slug=$(basename "$project_root" \
  | tr '[:upper:]' '[:lower:]' | tr -c 'a-z0-9' '-' \
  | sed 's/--*/-/g;s/^-//;s/-$//')
ts=$(date +%Y-%m-%d-%H%M)
out_dir="$HOME/.claude/handovers/$project_slug"
mkdir -p "$out_dir"
```

## Decide: update existing, or create new?

A long task often spans many `/handover` calls. Treating each one as a fresh doc fragments context. So the skill **prompts the user** in the ambiguous case, and uses sensible defaults otherwise.

### Decision table

| Situation | Action |
|-----------|--------|
| PreCompact hook invocation (marker file present) | Create new, label `auto`. No prompt. |
| Label provided + matches existing file in this project | Update that file. No prompt. |
| Label provided + no match | Create new with that label. No prompt. |
| No label + no existing handovers in project | Create new with auto-slug. No prompt. |
| **No label + existing handovers in project** | **Prompt user** (see below). |

### The prompt

List the existing handovers first so the user has context. Then use AskUserQuestion with these options:

1. `Update most recent: <slug> (iter N → N+1)` — picks the newest file by mtime
2. `Create new handover` — fresh doc with auto-generated slug from the current session
3. `Update a different existing one` — when picked, print the numbered list and wait for the user's next message (a number or slug)

If the user picks option 3 and types a number/slug, resolve it via the same exact→fuzzy matching used by `/handover-resume`.

### Auto-slug rules (only when creating new)

Claude generates a 3-5 word kebab slug from session content. Lowercase, hyphens only, no trailing/leading dashes. Final filename: `${ts}-${slug}.md`.

## Writing the doc

Use the Write tool with the template below. **Be tight but complete.** Write only what would save the next session real time — quality over quantity. The two sections that should NOT be paraphrased or condensed are **Verbatim quotes** (loss-prevention) and **Commands and artifacts** (concrete recoverable values).

For each other section: if a section truly has nothing useful, write `None.` and move on — don't pad.

### Template

```markdown
---
project: <project_slug>
cwd: <pwd>
branch: <current git branch, or "no-git">
created: <ISO 8601 UTC>
label: <slug>
session_id: <CLAUDE_SESSION_ID if set, else empty>
---

# Handover: <Short human title>

## Goal
One sentence — what we were doing and why.

## Context a fresh session needs
2–4 sentences max. Background not derivable from code or git history:
constraints, prior decisions outside this session, related work.

## Current state
3–6 concrete bullets. Done / in-flight / blocked.

## Timeline of what I tried
Bullets, chronological. Include dead-ends only if a future session might
otherwise re-attempt them. Skip this section entirely (write `None.`) if
the path was linear.

## Key decisions and rationale
3–5 bullets. Each: the decision + the rejected alternative + 1-line reason.

## Verbatim — user corrections and preferences
Quote (do not paraphrase) every time the user pushed back, corrected, or
expressed a preference. Generous here — paraphrasing is the biggest loss
vector.
> "..."

## Open questions
Bullets. Things unresolved.

## Next steps (prioritized)
Numbered. Short imperative phrasings.
1. ...

## Important files
Bullets. `path:line` — one-line reason it matters.

## Loose ends
Bullets, or `None.` Only things mentioned in the session that don't fit
above but might matter later.

## Commands and artifacts
Bullets. Concrete and verbatim — these are the most recoverable values.
- Commands that worked (with flags)
- Error messages encountered, with the fix that resolved them
- URLs, ticket IDs, PR numbers, log paths
```

## Workflow

1. **Compute paths** — run the bash above to get `out_dir` and `ts`.
2. **List existing handovers** in `$out_dir` newest-first (by mtime: `ls -1t`).
3. **Decide** create vs update per the decision table above. If the case calls for a prompt, ask via AskUserQuestion now, before doing anything else.
4. **Resolve the target file:**
   - **Update path:** the chosen existing file (keep its filename).
   - **Create path:** `${ts}-${slug}.md` where slug = user arg, or `auto` for hook, or auto-generated.
5. **Write the doc** — see "Create" or "Update" sections below.
6. **Confirm** — one line: `Saved: <full path> (iter <N>)` for updates, `Saved: <full path>` for new docs.

### Create (new doc)

Use the Write tool. Template above. Frontmatter:

```yaml
---
project: <slug>
cwd: <pwd>
branch: <git branch or "no-git">
created: <ISO 8601 UTC, current time>
updated: <same as created>
iteration: 1
label: <resolved slug>
session_id: <CLAUDE_SESSION_ID if set, else empty>
---
```

### Update (existing doc)

A handover is **compounding** — some sections roll/append (preserving history), others snapshot (replace with latest).

Steps:

1. **Read** the existing file with Read.
2. **Parse frontmatter.** Note `created`, `iteration` (default 1 if missing).
3. **Build the new content:**

   | Section | Update mode |
   |---------|-------------|
   | Goal | Keep as-is. Only rewrite if scope genuinely changed (rare) |
   | Context a fresh session needs | Keep existing. If new constraints emerged, append a subsection: `### Update (iter N, YYYY-MM-DD HH:MM)` with new notes |
   | Current state | **Replace** with latest snapshot |
   | Timeline of what I tried | **Append** a new `### Iteration N (YYYY-MM-DD HH:MM)` subsection at the end with this iteration's attempts/outcomes |
   | Key decisions and rationale | **Append** new decisions (group under `### Iteration N` if there were any new ones) |
   | Verbatim — user corrections and preferences | **Append** new quotes from this iteration. Never delete old ones. |
   | Open questions | **Replace** with currently-open list (drop resolved ones) |
   | Next steps (prioritized) | **Replace** with current pending steps |
   | Important files | **Replace** with current latest list |
   | Loose ends | **Append** if any new ones. Keep existing. |
   | Commands and artifacts | **Append** new entries. Keep existing. |

4. **Update frontmatter:**
   - `updated`: current ISO timestamp
   - `iteration`: bumped by 1
   - `created`, `label`, `project`: unchanged
5. **Write back** with Write tool (full file).

## Notes for the model

- **Single Write call.** Don't iterate or expand. One pass.
- **No extra tool calls** to flesh out sections. If you don't already know it from the conversation, leave it out — the next session can re-derive it.
- **Verbatim quotes are NOT optional** when the user pushed back, corrected, or stated a preference. Quote with `>`. Paraphrasing is the biggest loss vector.
- **Important files**: include `path:line` when relevant.
- **Don't duplicate git/code state** — the next session has access to `git log`, `git diff`, and the files. The handover supplements those, never duplicates them.

## Performance tip for the user

This skill runs in the active session, so it uses the session's effort level. On a long conversation at `effortLevel: xhigh` or `max`, a single `/handover` invocation can take several minutes. If you want a faster manual handover, type `/effort medium` (or `low`) before running it — the trade is a bit less polish in the writing, not loss of structure or required content.

The PreCompact auto-handover runs at `--effort high` for the sub-Claude call, independent of session effort.
