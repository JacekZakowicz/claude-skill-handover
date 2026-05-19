# claude-skill-handover

Two Claude Code skills for session continuity:

- **`/handover`** — saves the current session's state to a rich markdown doc so a future session can pick up the work with full context.
- **`/handover-resume`** — loads a previously-saved handover and continues the work, with fuzzy search across past handovers.

Built for the workflow of long tasks that span multiple `/clear`s, context compactions, or multi-day work — without losing decisions, user corrections, or in-flight state.

## What you get

### `/handover` — save session state

Saves a structured markdown doc capturing:

- **Goal, context, current state** — what we were doing and where we are
- **Timeline + key decisions** — what was tried, what was chosen, and why
- **Verbatim user quotes** — every correction, pushback, or preference, quoted exactly (not paraphrased)
- **Next steps, open questions, important files** — the runway for the next session
- **Commands and artifacts** — concrete recoverable values: working commands, error→fix pairs, PR/ticket URLs

### `/handover-resume` — load and continue

Picks up a saved handover and resumes work:

- No arg → asks which handover to resume (or auto-picks the newest)
- `/handover-resume 2` → loads the 2nd-newest
- `/handover-resume port-collision` → exact substring match, then fuzzy fallback via `fzf`
- Flags branch/cwd drift if the current workspace doesn't match the saved one

## Key features

- **Compounding handovers** — re-running `/handover` on the same task updates the existing doc (bumps `iteration`, appends to history sections, replaces snapshot sections) instead of creating a new file each time. Long tasks stay coherent across many sessions.
- **Worktree-aware project slug** — handovers are keyed off the **main** worktree's basename, so all worktrees of the same repo share one handover folder. Switching worktrees doesn't fragment your history.
- **PreCompact auto-save** — when the Claude Code PreCompact hook is wired up, `/handover` runs automatically before context compaction (with `label: auto`), so you never lose state to a compaction boundary.
- **Fuzzy resume** — `/handover-resume <query>` tolerates typos and partial recall (`prtcol` resolves to `port-collision`) when `fzf` is installed.
- **Global storage, never in the repo** — handovers live in `~/.claude/handovers/`, so they survive branch switches, repo deletes, and don't pollute git.

## Storage layout

```
~/.claude/handovers/
  <project-slug>/
    2026-05-18-1432-empty-state-projection-fix.md
    2026-05-18-2210-checkout-flow-refactor.md
    2026-05-19-0904-debug-ci-build.md
```

`<project-slug>` is the basename of the main git worktree (or `cwd` if not a git repo), lowercased and kebab-cased.

## Installation

Add this repo as a marketplace in your Claude Code settings. In `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "claude-skill-handover": {
      "source": {
        "source": "github",
        "repo": "JacekZakowicz/claude-skill-handover"
      }
    }
  },
  "enabledPlugins": {
    "claude-skill-handover@claude-skill-handover": true
  }
}
```

Then restart Claude Code. The `/handover` and `/handover-resume` slash commands will be available.

### Optional: PreCompact hook for auto-save

To auto-save a handover before context compaction, add a PreCompact hook to your Claude Code settings that invokes the `handover` skill. The skill detects hook invocations and creates a new handover with `label: auto`, no prompt.

### Optional: install `fzf` for fuzzy resume

```bash
brew install fzf
```

Without `fzf`, exact substring matching still works — fuzzy is just a nicety for typos and partial recall.

## Usage

### Saving a handover

```
/handover
```

If there are no existing handovers for this project, creates a new one with an auto-generated slug from the session content.

If there are existing handovers, asks whether to update the most recent (bump `iteration`), create a new one, or update a different one.

```
/handover checkout-flow
```

Skips the prompt. If a handover with `checkout-flow` in the slug exists, updates it. Otherwise creates a new one with that label.

### Resuming a handover

```
/handover-resume
```

Prints the recent handovers list and asks which to load.

```
/handover-resume 1
```

Loads the most recent (1st in the list).

```
/handover-resume checkout
```

Exact substring match against handover filenames; falls back to fuzzy match if no substring hit.

## Why this exists

`/clear` and context compaction lose decisions, user corrections, and in-flight state. Git history captures *what* changed but not *why* — and definitely not the verbatim "no, don't do it that way" moments that should never be re-litigated.

The handover doc is a compounding artifact: each iteration adds to the timeline and verbatim-quotes log without overwriting them, while snapshot sections (current state, next steps, open questions) reflect the latest reality. The result is a single file that a fresh session can read and immediately be caught up.

## License

MIT
