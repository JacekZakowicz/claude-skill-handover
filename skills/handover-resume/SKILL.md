---
name: handover-resume
description: Resume a previous Claude session by loading a handover document saved by /handover. Use when the user types /handover-resume [number-or-slug] or asks to "resume", "continue last session", "pick up where I left off", or "load the handover". Auto-loads the newest handover for the current project unless they specify a different one.
---

# Handover Resume

Load a previously-saved handover document and continue the work it describes.

## Locate handovers

```bash
# print everything after "worktree " so paths containing spaces survive intact
project_root=$(git worktree list --porcelain 2>/dev/null | awk '/^worktree /{sub(/^worktree /,"");print;exit}')
[[ -z "$project_root" ]] && project_root=$(pwd)
project_slug=$(basename "$project_root" \
  | tr '[:upper:]' '[:lower:]' | tr -c 'a-z0-9' '-' \
  | sed 's/--*/-/g;s/^-//;s/-$//')
handover_dir="$HOME/.claude/handovers/$project_slug"

# Newest first
ls -1t "$handover_dir"/*.md 2>/dev/null
```

If the directory is empty or missing, print `No handovers for <project_slug>` and stop. Do not proceed.

## Resolve which handover to load

User argument forms:

- **No arg** — load the newest (top of `ls -1t`).
- **Numeric** — `/handover-resume 2` loads the 2nd-newest.
- **Slug substring or fuzzy** — `/handover-resume port-collision` tries an exact substring match first. If none, falls back to fuzzy via `fzf --filter`, which tolerates typos and partial recall (`prtcol` or `port-collison` both resolve to `port-collision`).

### Matching algorithm

Portable across bash and zsh (no arrays, just newline-separated strings and pipes):

```bash
# 1. Exact substring (fast path) — preserve newest-first order
exact=$(ls -1t "$handover_dir"/*.md 2>/dev/null | grep -F -- "$query")

if [[ -n "$exact" ]]; then
  n_exact=$(printf '%s\n' "$exact" | wc -l | tr -d ' ')
  chosen=$(printf '%s\n' "$exact" | head -1)
  # If n_exact > 1, the chosen one is already the newest.
elif command -v fzf >/dev/null 2>&1; then
  # 2. Fuzzy fallback via fzf --filter
  fuzzy=$(ls -1t "$handover_dir"/*.md 2>/dev/null | fzf --filter "$query")
  if [[ -n "$fuzzy" ]]; then
    n_fuzzy=$(printf '%s\n' "$fuzzy" | wc -l | tr -d ' ')
    if [[ "$n_fuzzy" -eq 1 ]]; then
      chosen="$fuzzy"
    else
      chosen="ASK"
      top_matches=$(printf '%s\n' "$fuzzy" | head -10)
    fi
  fi
fi
```

If fzf isn't installed and there's no exact match, fall through to "Zero matches".

### Behavior by outcome

| Outcome | What to do |
|---------|-----------|
| Single exact substring match | Load it. Note: `Matched "<query>" exactly.` |
| Multiple exact substring matches | Load newest. Note: `<N> matched "<query>"; loading newest.` |
| Single fuzzy match | Load it. Note: `Fuzzy-matched "<query>" → <slug>.` Use the slug part of the filename, not the full path. |
| Multiple fuzzy matches | Print top 10, ask user to pick a number. Do not load anything yet. |
| Zero matches anywhere | Print the full list and ask user to clarify. Do not load anything. |
| fzf not installed and no exact match | Print the full list (newest first) and ask user to clarify. |

## Workflow

The list is **always** printed for transparency. When `/handover-resume` is called with no arg and there are existing handovers, the skill **asks** the user which to load rather than auto-picking. With an arg, behavior is rule-driven (exact / fuzzy / ambiguous).

1. **List** — gather all `.md` files in the handover dir, newest first. Build a numbered list `<N>. <HH:MM> <today|yesterday|N days ago> — <slug> (iter <M>)`. Pull `iter` from the frontmatter if present.
2. **Print the list** — always, regardless of args. Format:

   ```
   Recent handovers for <project_slug>:
     1. 09:04 today    — empty-state-simplify-projection-loop-fix (iter 2)
     2. 00:19 today    — checkout-flow-refactor (iter 5)
     3. 18:44 yesterday — debug-ci-build
   ```

3. **Resolve the pick:**

   **A. No arg + handovers exist → ASK** via AskUserQuestion with three options:
   - `Resume most recent: <slug>`
   - `Pick a different one` (then wait for user's next message — they type a number or slug)
   - `Just show the list, don't load` (terminate the skill, leave the printed list as the response)

   **B. Arg provided → resolve via the matching algorithm above:**
   - Single exact substring match → that file (auto-load)
   - Multiple exact matches → newest of them (auto-load)
   - Single fuzzy match → that file (auto-load, note the resolution)
   - Multiple fuzzy matches → list top 10 + ask user to pick a number (STOP, wait)
   - Zero matches → full list already printed above, ask user to pick a number (STOP, wait)
   - Numeric arg out of range → say `Only <M> handovers — pick 1-<M>` (STOP, wait)

4. **Load** (when a pick is resolved) — Read the chosen file in full.
5. **Brief** — print a one-line header:
   `Loaded: <filename> (iter <N>, last updated <relative age>) — picking up at "Next steps"`.
   If a fuzzy match was used, include the resolution on the same line.
6. **Continue the work** — internalize the doc as your briefing. Start from the "Next steps" section. Treat everything in the doc as established context for this session.

## After loading

Do NOT regurgitate the doc back at the user. They wrote `/handover-resume` because they want to continue, not re-read what they already know. Print only the one-line header, then either:

- Ask the user how they want to proceed if the next steps need a decision, or
- Start executing the first concrete next step if it's unambiguous.

## Notes

- Time formatting: use "today" / "yesterday" / "<n> days ago" for human-friendliness.
- The doc's frontmatter `branch` field may differ from the current branch. If so, flag it: `Loaded handover was on branch <X>, current branch is <Y> — confirm before proceeding`.
- If the handover's `cwd` differs from current `pwd`, also flag it. Cross-worktree resume is possible but worth confirming.
