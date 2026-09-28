---
name: prep
description: Prime a work session and create a durable, compaction-surviving state file. Run ONCE at the start of a terminal session before doing real work. It reads the code state and project rules (CLAUDE.md / AGENTS.md / README), then writes a lean session-state file under .claude/prep/ that records code state, rules digest, goal, key decisions, open forks, and a clean loop log. This file is re-read after every compaction so context is never lost. Invoke when the user types /prep, or before starting an orchestrate-loop in a fresh session.
---

# Prep

Builds a **durable session-state file** so a long session survives compaction without
losing the thread. Claude's own session transcripts are bloated and get summarized
away; this is a small, hand-curated file on disk that you re-read to recover state.

Run `/prep` **once at the start of a terminal session**, before real work. The
orchestrate-loop (and you, generally) then read and update this file as you go.

## Where the file lives

`<project>/.claude/prep/session-<YYYY-MM-DD-HHMM>.md`

- Project-local `.claude/prep/`, NOT `/tmp`, NOT the Claude session transcript dir.
- The **active prep file is the most recently modified `session-*.md`** in that dir.
  That convention is how you find it again after a compaction wipes your memory of
  the path.
- It is **valid only for the current terminal session.** A fresh terminal = run
  `/prep` again = a new dated file. (You can't always read a terminal-session id from
  the environment, so treat "latest dated file + a new `/prep` per session" as the
  rule.)
- Ensure `.claude/prep/` is gitignored (add it if not) — this is session scratch,
  never committed.

## What `/prep` does

1. **Read the code state** — branch, last commit(s), working-tree status, whether it
   builds / tests pass (a quick check, not a heavy run), project layout at a glance.
2. **Read the rules** — CLAUDE.md, AGENTS.md, README, and any contributing/landmine
   docs. Distill the *binding* constraints, not the prose.
3. **Establish the goal** — what this session is for (ask the user if unclear, or
   infer from their first request).
4. **Write the prep file** using the template below. Keep it LEAN — key items only.
5. Tell the user it's written and where.

## Prep file template

```markdown
# Session — <YYYY-MM-DD HH:MM> — <project name>

Terminal-session-scoped working state. Re-read this after every compaction.
Last updated: <timestamp>

## Goal
<one or two lines: what this session is trying to achieve>

## Code state
- branch: <branch> @ <short-sha> "<subject>"
- build/tests: <e.g. "22 tests pass, clippy clean" or "not yet checked">
- working tree: <e.g. "go.rs + javascript.rs untracked; mod.rs modified">
- layout: <one-line orientation if useful>

## Rules digest (binding constraints)
- <key rule 1 — e.g. "no storage/IO in core; pure deterministic">
- <key rule 2 — e.g. "publishing discipline: no internal names in shipped code">
- commands: <build/test/lint commands>

## Key decisions (durable)
- <decision + one-line why — e.g. "Java namespaces from package decl, not path">

## Open forks (undecided)
- <question awaiting a decision, or "none">

## Loop log (terse — forward-looking, not an archive)
- ✅ <completed+committed units> — collapse to a count or one short line; git holds the detail
- 🔄 <in-progress unit> — <current step> (the ONE thing that needs full detail)
- ⏭️ <queued next units> — one line each

## Obsolete (cleared this update)
<remove stale lines entirely; this section is just a reminder to prune, keep it empty>
```

## Maintenance rules (apply on every update, not just at creation)

- **Forward-looking, not a retrospective.** The prep file exists to help the NEXT
  session continue — it holds only what a future you needs to resume: current state,
  what's left, durable decisions/rules, open forks. It is NOT a log of what was
  accomplished. Anything already finished, committed, and needing no future reference
  does not deserve detail — **git history is the archive; the prep file is the map.**
- **Collapse completed work.** A finished+committed unit becomes ONE terse line (or
  folds into a running count like "✅ 12/14 done: <list>"). Don't preserve each unit's
  blow-by-blow (the node kinds, the bug you hit, the hashes) — that detail is in the
  commits. Keep full detail for exactly ONE thing: the in-progress unit, if any.
- **Lean, real, current.** Key items, key forks, key decisions only. Not a transcript.
  If a line no longer matters, **delete it.** A stale or bloated prep file is worse than
  a short one — verbosity buries the few facts that actually matter for resuming.
- **Re-read your own prep critically.** When you refresh it, if it's grown past ~a
  screen, that's a smell — prune completed detail until only the map remains.
- **Prune before you append.** Each update: remove done/abandoned/superseded lines,
  then add the new fact.
- **Decisions vs forks.** When a fork is resolved, move it to Key decisions with the
  rationale and delete the fork line.
- **Flush before risk.** Always update + save the file BEFORE anything that could
  reset context (a compaction, a long loop iteration, handing off). On disk = safe.
- **One file per session.** Don't spawn extras; update the active one in place.

## Relationship to other skills

The `orchestrate-loop` skill depends on this file: before each loop iteration it
re-reads the active prep file, refreshes it, and checks context headroom. If no prep
file exists when a loop starts, create one first (run this skill's steps).

