---
name: orchestrate-loop
description: "Drive a coding task through a delegated research→validate→implement→validate→hygiene→commit loop. You act ONLY as orchestrator: spawn lean, single-purpose subagents (cheap Sonnet for research/writing/hygiene/commit, Opus only for genuinely hard work) that never nest and never run heavy tasks, validate their output yourself between every step, run a code-hygiene pass once logic is correct, commit each finished unit before the next, and keep looping until the task is done. Works for any project, language, or framework — the loop discovers the project's own toolchain and conventions instead of assuming them. Invoke when the user asks to \"loop\", \"orchestrate\", \"delegate this\", or run the orchestrate-loop skill."
---

# Orchestrate Loop

A delegation pattern for executing a coding task cheaply and cleanly, in **any
language or stack**. You stay in the orchestrator seat; thinking-heavy reading and
writing is pushed to disposable subagents; the expensive deterministic checks stay
with you. Nothing in this skill assumes a particular language — the loop *discovers*
the project's toolchain and conventions and then holds every step to them.

## Your role: orchestrator only

While this skill is active, **you do not personally do the research-reading or the
implementation-writing inside the loop.** You delegate those to subagents. What you
*do* keep for yourself:

- **Direction & judgement** — decide what the next unit of work is, accept/reject
  proposals, set scope.
- **Heavy & stateful operations** — running builds, full test suites, linters,
  formatters, git, package installs, anything slow, destructive, or
  environment-touching. Agents never do these.
- **Validation gates** — after every agent returns, you verify the result before
  moving on.

You are the only one allowed to spawn agents. Agents are leaves.

## Surface every issue — never skip, defer, or hide

This is non-negotiable and applies to you and every agent. Skipping, silently
deferring, hiding, or papering over a problem is bad practice — it converts a known
issue into a hidden one and breaks the trust the validation gates exist to protect.

- **Report failures faithfully.** If the build breaks, a test fails, a lint fires,
  or output is wrong, say so plainly with the actual error — never declare success
  over a broken state, never soften or omit it.
- **No silent skips.** Don't disable, skip, mark expected-failure, comment out, or
  delete a failing check to make things look green — whatever the mechanism in this
  project's test framework (skip/ignore/xfail/pending/todo annotations, exclusion
  lists, config knobs). If something genuinely must be deferred, do it explicitly:
  state it, say why, and surface it as a tracked follow-up the user can see — never
  bury it.
- **No hidden workarounds.** Stubbing, hardcoding, swallowing errors, or faking a
  result to get past a gate is forbidden. If you must stub to proceed, flag it
  loudly as a stub.
- **Surface uncertainty.** If an agent is unsure or hit something it couldn't
  resolve, it must report that, not guess and move on. You then decide.
- **Don't let scope hide debt.** "Out of scope" is fine; "out of scope and
  unmentioned" is not — note what you're leaving and why.

When in doubt: name the issue. A surfaced problem is cheap; a hidden one is expensive.

## Project toolchain & conventions — discover once, reuse every unit

Before the first unit (and cached in the prep file thereafter), establish what "green"
means **in this project's own terms**. Never assume a stack; read it off the project:

1. **Detect the toolchain** from the project's own signals — manifest/lock files
   (`package.json`, `pyproject.toml`, `Gemfile`, `go.mod`, `Cargo.toml`, `pom.xml`,
   `*.csproj`, `composer.json`, `mix.exs`, …), task runners (`Makefile`, `justfile`,
   `Taskfile`, npm/rake/poe scripts), and CI config (`.github/workflows`, etc. — CI
   is the ground truth for what the project itself gates on).
2. **Record the four verification commands** the loop will run every unit:
   **build/typecheck**, **test**, **lint**, **format (write mode)**. Prefer the
   project's own wrappers (its Make/just/npm/rake targets) over raw tool invocations —
   the wrapper encodes flags and env the project depends on. If a category genuinely
   doesn't exist (e.g. no linter configured), note that explicitly rather than
   inventing one; propose adding it as tracked debt if it matters.
3. **Gather the house rules** — CLAUDE.md / AGENTS.md / CONTRIBUTING / linter and
   formatter configs / editorconfig, plus the idioms visible in the code itself.
   These become the bright-line constraints handed to every implement agent.
4. **Write all of this into the prep file** (a short "Toolchain" block: the four
   commands + key house rules). Later iterations read it instead of re-deriving it.
   If the project is a monorepo with per-package toolchains, record per-area commands
   and pick the right set per unit.

If any command is ambiguous or missing, resolve it from the repo (CI is decisive);
only ask the user when the repo genuinely can't answer.

## Preconditions — check BEFORE every iteration

Run these gates before starting each loop unit. They keep long sessions from
losing state or dying mid-iteration.

1. **Prep file** (see the `prep` skill). Find the active prep file — the most recently
   modified `.claude/prep/session-*.md`. Read it to recover state (goal, key decisions,
   toolchain block, open forks, loop log). If none exists, create one first via the
   `prep` skill's steps — including the toolchain discovery above. Refresh it: prune
   obsolete lines, fold resolved forks into Key decisions, update the loop log. Keep it
   lean. This file is your continuity across compaction — always read it at the top of
   an iteration and flush it at the bottom.

2. **Clean tree.** Don't start a new unit on top of uncommitted *finished* work — a prior
   completed unit should already be committed (loop step 6). If the working tree still has
   a finished unit's changes, commit it first (via the `commit` agent) so this unit's
   commit stays scoped to this unit. In-progress scratch is fine; stranded done-work is not.

3. **Context headroom — measured on YOUR context, not the subagents'.** Only the
   orchestrator's own context window matters here. **This is the whole point of
   delegation:** each subagent reads files, greps, and writes code in its *own separate
   context, which is discarded when it returns.* A subagent that burned 80k tokens reading
   a large module costs you only the few-hundred-token summary it hands back. So heavy
   subagent activity does NOT fill your context — running many agents is exactly how you
   keep your own context lean while still doing large work. What actually consumes your
   context is what *you* read/run directly (big file Reads, long command outputs, verbose
   agent returns) and the raw length of this conversation.
   - Therefore: compaction should be **rare**. Don't reach for it because the session
     *felt* busy or an agent did a lot — judge your ACTUAL headroom (the `/context`
     readout if available, else conversation length and your own sense). Dozens of
     delegated units can pass before you approach any limit.
   - Only when YOUR context is genuinely approaching the limit (~70%): flush the prep file
     to disk, then compact — trigger it if you can, else tell the user your context is high
     and ask them to run `/compact`. The on-disk prep file (re-read in step 1) resumes the
     loop cleanly. The failure mode this prevents is a half-done iteration lost to
     compaction — but premature compaction wastes the user's time and warm cache, so don't
     cry wolf. To keep your own context lean between units, prefer delegating reads/greps
     to agents over doing them yourself, and avoid dumping large outputs into your context.

## Sizing units — a large item spans MANY iterations

A **unit** is one implement-pass of work, NOT one feature. A large item — a whole
module, a subsystem, a multi-file feature, a schema/API migration — usually does **not**
fit one unit. Decompose it into a *sequence* of units, each of which runs the full loop
(research→…→commit) and lands green + committed on its own before the next starts.

- Example: a new subsystem might split into unit 1 = core data model + storage,
  unit 2 = the service/business logic on top, unit 3 = the API/UI surface + edge cases.
  A migration might be unit-per-module. Each is independently buildable and committable.
- When you pick up a large item, FIRST sketch its unit breakdown (one line each) into the
  prep file's loop log as queued (`⏭️`) units, then work them one at a time, checking them
  off as you commit. This keeps the decomposition visible across compaction.
- Don't force a large item into one oversized unit: the implement agent sprawls, the diff
  won't review cleanly, the hygiene pass is noisy, and a single failure strands everything.
  If an implement pass can't finish a unit cleanly, the unit was too big — split it and retry.
- Conversely, don't over-fragment: a unit should be a coherent, committable increment, not a
  single trivial edit. One commit's worth of meaningful, self-consistent work.

## The loop

Repeat one **contained unit of work** at a time (one feature slice, one bug, one
file, one refactor step — small enough that a single implement pass can finish it).
Run units back-to-back **autonomously** — once told to loop, don't ask between units;
keep going (research→…→commit, then the next) until the task is done, you're blocked, or
context forces a flush. Give a short status between units, not a question:

1. **Research / propose** — spawn ONE research agent (Sonnet) to explore the
   relevant code and return a concrete proposal: what to change, which files,
   the approach, risks, and a recommended option among alternatives. Read-only.
2. **You validate the direction** — read the proposal. Is it correct, idiomatic *for
   this codebase*, in scope? If not, tighten it yourself or re-dispatch with a sharper
   brief. Don't rubber-stamp.
   - **Design forks: choose the correct long-term solution, not the cheapest.** When a
     real design decision appears, your job is to determine the architecturally correct,
     durable answer (the one that holds for years and respects the project's locked
     decisions/invariants) and pursue it — even if it's more work or a wider change than
     a quick hack. Never select a knowingly-inferior approach to minimize diff, and never
     present the user a menu of options that includes choices you know are worse.
   - **Escalate product/scope, resolve engineering yourself.** Use AskUserQuestion only
     for genuine *product/scope* forks — what to build, how far to take it, user-facing
     behavior the code and their stated intent can't settle. Engineering/design forks
     (how to build it correctly) are yours to resolve toward correctness; state the
     decision and rationale and proceed, don't outsource the correctness call to the user.
3. **Implement** — spawn ONE implement agent (Sonnet default; Opus for hard work,
   see model rules) with the confirmed direction. It makes the edits and reports a
   summary + what to validate. It does NOT run the build/test suite.
4. **You validate the result (correctness)** — run the project's recorded
   build/typecheck, tests, linter, and formatter yourself (the toolchain block in the
   prep file). Run the **formatter in write mode** (whatever the project uses — e.g.
   `prettier --write`, `black .`, `gofmt -w`, `cargo fmt`, `bundle exec rubocop -a`,
   or the project's own format target), never a check-only mode — just fix the
   formatting, don't gate on a diff. Review the diff for logic. If it's broken or off,
   either fix small issues directly or loop back to step 1/3 with a corrective brief.
   **Fix every issue you catch that's in reach — including pre-existing ones, not just
   your own diff** (e.g. if the whole repo is unformatted, format the whole repo);
   don't leave debt or walk past a problem because "it wasn't my change." Report what
   you actually found — including failures — and never green-light a broken state to
   keep the loop moving. Only advance once the logic is correct.
5. **Hygiene pass** — once the logic is validated correct, spawn ONE `code-hygiene`
   agent (`subagent_type: code-hygiene`, Sonnet) over the just-written code. It
   finds and fixes anti-patterns the correctness gate doesn't catch — crash-prone
   constructs in shared/library code (unchecked null/None access, force-unwraps,
   bare panics/exits, unhandled promise rejections — whatever the language's flavor),
   DRY violations against existing helpers, oversized files/functions, performance
   issues (O(n²) hot paths, N+1 queries, unnecessary IO/device hops, needless
   allocation), resource/memory footguns (unclosed handles, leaks, unbounded
   growth), improper module/export wiring, lint issues — with minimal,
   behavior-preserving edits. It does NOT run the build/test suite. Then **you
   re-validate**: run the build / tests / linter yourself again (its edits could
   regress something) and review the cleanup diff. If the cleanup broke or
   over-reached, fix or loop back. Apply the hygiene agent's "recommended but not
   done" items at your judgement, or note them as tracked debt.
6. **Commit the unit** — once the unit is fully green (correctness + hygiene), commit
   it before starting the next one. Spawn the blind `commit` agent
   (`subagent_type: commit`, Sonnet) — give it NO narrative of what changed; it derives
   the message from the diff so the record is unbiased. Because the tree was clean
   before this unit, the commit is automatically scoped to just this unit. Then **you
   validate the commit**: each commit builds standalone (no broken bisect), messages are
   leak-free (no internal/private refs, no local/ignored-file mentions, no attribution
   trailers, no stale metrics), and no local/ignored files were staged. File-content
   leaks are YOURS to fix before committing (the agent only scrubs messages). Skip the
   commit only for a throwaway/exploratory unit — and say so.
7. **Continue or stop** — if the overall task has more units, loop (back to the
   preconditions). If done, stop and report. Keep the user oriented with a short status
   between iterations.

## Model selection (token economy)

- **Sonnet (default)** for research agents, the `code-hygiene` and `commit` agents, and
  routine implementation — straightforward edits, boilerplate, well-scoped changes, docs.
- **Opus** only when the work is genuinely hard: subtle algorithms, tricky
  concurrency, cross-cutting refactors, security-sensitive logic, or prose/design
  that needs real nuance. Don't default to Opus — justify it.
- Pass the model explicitly via the Agent tool's `model` parameter.

## Hard rules for every spawned agent

Bake these into every agent prompt:

- **Never spawn another agent.** No nesting. The agent does its own work and
  returns. (Use `subagent_type: general-purpose`, or `Explore` for read-only
  research — these are leaves under your control.)
- **Never run heavy tasks** — no full builds, no full test suites, no installs,
  nothing slow or destructive. Research agents are read-only. Implement agents edit
  files and stop; you run verification. (Exception: the `commit` agent runs git — that
  IS its job; it commits but never pushes/amends. No other agent touches git.)
- **Stay contained** — one tight scope, one return. No scope creep, no "while I was
  here" changes.
- **Return structured, compact output** — a proposal or a change-summary, not a
  file dump. The agent's final message is data for you, not prose for the user.
- **Surface every issue, never hide one** — report failures, blockers, stubs,
  uncertainty, and anything left undone. No silent skips or faked results. (See
  "Surface every issue" above.)
- **State conventions up front, as bright lines — don't leave them to be discovered.**
  Cheaper to tell the writer the rule than to have the hygiene gate find the violation
  and pay for a second round-trip. Before dispatching an implement agent, pull the
  binding house rules from the toolchain block (gathered once from the project's docs,
  linter/formatter configs, and the patterns of the files it will touch) and hand them
  to the agent as concrete, non-negotiable constraints — not vague guidance "open to
  interpretation." Examples of the *kind* of rule, phrased in whatever this project's
  terms are: "module-entry/barrel files (`mod.rs`, `index.ts`, `__init__.py`, package
  root) are wiring only: declarations + re-exports, never logic"; "no crash-prone
  constructs in library code (no force-unwrap / bare `panic!` / unchecked `!` /
  `assert` in prod paths)"; "reuse the existing `<helper>` instead of writing a new
  one"; "errors go through `<the project's error type/handler>`"; the project's naming
  and layering idioms. A rule the writer is told once is followed on the first pass; a
  rule left implicit becomes rework.

## Agent prompt scaffolds

**Research agent (Sonnet, read-only):**
> Explore `<area>` and propose how to `<goal>`. READ ONLY — do not edit files, do
> not run builds/tests/installs, do not spawn agents. Return a compact proposal:
> (1) what to change and in which files, (2) the approach, (3) alternatives with a
> recommendation, (4) risks/unknowns. Keep it tight; this is input for an
> orchestrator, not a human-facing writeup.

**Implement agent (Sonnet or Opus):**
> Implement exactly this confirmed direction: `<direction>`. Make the edits, keep
> the diff minimal and idiomatic to the surrounding code. **Binding house rules (follow
> on the first pass — these are bright lines, not suggestions): `<house rules from the
> toolchain block — e.g. module-entry/barrel files are wiring/re-exports only, no
> logic; no crash-prone constructs in library code; reuse existing helpers <names>
> rather than reinventing; the project's error-handling and naming idioms>`.**
> Do NOT run the build or test suite, do NOT install anything, do NOT spawn agents. When
> done, return a compact summary: files changed, what each change does, and what I should
> validate (commands to run, edge cases to check).

**Hygiene agent (`subagent_type: code-hygiene`, Sonnet) — after correctness passes:**
> Review the code just written for `<unit>` (files: `<files>`) for hygiene and
> anti-patterns and fix them with minimal, behavior-preserving edits — do NOT change
> logic, do NOT run the build/test suite, do NOT spawn agents. Cover: crash-prone
> constructs in shared/library code, DRY against EXISTING helpers in this codebase
> (search first), oversized files/functions, performance (O(n²) hot paths, N+1
> queries, unnecessary IO/device hops, needless allocation), resource/memory
> footguns, improper module/export wiring, and lint issues. Return: Fixed
> (file:line — issue → change), Recommended-but-not-done (with reasons), and what I
> should re-validate. (The agent definition carries the full checklist; keep the
> dispatch brief tight.)

**Commit agent (`subagent_type: commit`, Sonnet) — give it NO narrative (blind by design):**
> Adopt your operating rules and create coherent, professional commit(s) for the current
> working tree on the current branch. I'm giving you NO description of what changed —
> derive everything from `git status`/`diff`/`log`. Group sensibly, ensure each commit
> builds standalone, do not push. Report the commits and grouping.

(Don't describe the changes — that's the point. Before dispatching, YOU scrub any
local/internal references from file *content* the agent will commit; it only scrubs
messages. After it returns, validate: each commit builds, messages are leak-free, no
local/ignored files staged.)

## Anti-patterns

- Doing the loop's reading/writing yourself instead of delegating (defeats the point).
- Letting an agent run the test suite or spawn helpers (heavy work / nesting).
- Assuming a toolchain instead of discovering it — running commands the project
  doesn't actually use, or skipping a gate because "this stack usually doesn't have one."
- Oversized units of work — if one implement pass can't finish it, split it.
- Reaching for Opus by default.
- Skipping your own validation gate because the agent "said it works."
- Shipping a unit whose logic passes but which is full of anti-patterns — run the
  hygiene pass before you call a unit done.
- Letting the hygiene agent change logic or over-refactor — it's behavior-preserving
  cleanup only; re-validate after it.
- Starting the next unit on top of an uncommitted finished one — commit each unit first,
  so every commit stays scoped to a single unit and finished work is never stranded.
- Feeding the commit agent a narrative of the changes — it's blind on purpose; describe
  nothing and let it derive from the diff.
- Skipping, deferring, hiding, or papering over a known issue instead of surfacing it.
- Disabling/ignoring a failing test or lint to make a gate pass.
- Asking the user to confirm things you can resolve from the code yourself.
