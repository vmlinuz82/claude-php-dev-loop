---
name: php-dev-loop
description: Use when implementing a feature, bugfix, or Jira ticket in any PHP project (modern Symfony or legacy alike), or when executing an implementation plan against a PHP codebase — work that should go through a subagent write → review → rework loop before being shown to the user
---

# PHP Dev Loop

Execute a task list: for each task, dispatch a fresh implementer subagent,
review that task's diff with a reviewer subagent, and loop fix → re-review
until the task is approved — then move to the next task. Finish with one
adversarial final review over the whole change. **Never commit** — the loop
ends at an approved, uncommitted diff the user commits themselves.

**Core principles:**

- **Small iterations.** A task is the smallest unit that carries its own
  test cycle and is worth a reviewer's gate. Reviewers see one task's diff,
  never the whole ticket's — small diffs get real scrutiny, big ones get
  skimmed.
- **Isolated context.** Implementer and reviewers never share context. You
  (the controller) pass artifacts as files, not pasted text, and never let
  an agent review its own work. One fresh implementer per task — never
  reuse an implementer across tasks.

**Continuous execution:** do not check in with the user between tasks or
iterations. Stop only for: BLOCKED you cannot resolve, genuine ambiguity,
or the loop completing.

## Roles and Models

The skill is self-contained: every subagent is dispatched as
`general-purpose` and adopts a role bundled in this skill's `agents/`
directory — no pre-installed agents required. Every dispatch prompt MUST
begin with:

> Read `.php-dev-loop/skill/agents/<role file>` and adopt that role
> completely before doing anything else. (Path is relative to the repo
> root — Setup copies the skill files there.)

| Role | Role file | Model | When |
|------|-----------|-------|------|
| Implementer | `agents/php-implementer.md` | `opus`; `fable` for architectural decisions or high-risk changes (payments, auth, data migrations) | One per task |
| Fixer | `agents/php-implementer.md` | `opus` | Review findings |
| Iteration reviewer | `agents/iteration-reviewer.md` | `opus`; `fable` when the task diff is large, subtle, or security-sensitive | After every implement/fix dispatch |
| Final reviewer | `agents/final-reviewer.md` | `fable` — always the most capable | Once, after the last task is approved |

All roles share `agents/coding-standards.md` — edit standards there, once.

Well-decomposed tasks are mostly `opus` implementer work — that is the
point of decomposing. **Always set `model:` explicitly on every dispatch.**
An omitted model inherits the session's model — often the most expensive —
which silently defeats this table. Escalation ladder is
`opus → fable`, never downgrade: a fixer facing a design-level or
architectural finding gets re-dispatched on `fable`; a model that reported
BLOCKED gets re-dispatched on the next tier up, never retried unchanged.
Turn count beats token price — `opus` is the floor for every dispatch,
including all reviews.

## Setup (before first dispatch)

Work from the repository the task targets. The scratch space is
`.php-dev-loop/` at the repo root — inside the working directory, so file
tools never leave the project. It is git-excluded during the loop and
deleted at the end.

1. **Hide scratch from git FIRST:** run `git rev-parse --git-path
   info/exclude` and append a `.php-dev-loop/` line to that file with the
   file tools (skip if already present; the Write tool creates the file
   and its parents in fresh repos). This must precede the baseline
   snapshot so scratch is never captured.
2. **Resume check:** `cat .php-dev-loop/progress.md`. If it exists, tasks
   marked approved there are DONE — trust the ledger over your own
   recollection (compaction erases conversation memory, not files), resume
   at the first task not marked approved. Otherwise create the ledger with
   the Write tool (this also creates the directory).
3. **Baseline-0 snapshot:** run these two commands exactly:
   ```bash
   GIT_INDEX_FILE=.php-dev-loop/snap-index git add -A
   GIT_INDEX_FILE=.php-dev-loop/snap-index git write-tree
   ```
   The first stages the entire worktree into a THROWAWAY index file inside
   scratch — the real index, worktree, and refs are untouched. The second
   prints a tree SHA: the exact worktree state, including the user's
   uncommitted and untracked files, so pre-existing changes are invisible
   to every reviewer. Write the SHA to `.php-dev-loop/baseline-0`.
4. **Copy the skill in:** `cp -r "<SKILL_DIR>/." ".php-dev-loop/skill/"`
   (`<SKILL_DIR>` = the directory containing this SKILL.md). You and every
   subagent read role files and prompt templates from this in-project
   copy, never from the plugin directory.

Keep every git command a plain `git ...` invocation — no variable
assignments, `$( )` captures into variables, or `{ }` groupings wrapping
them; capture outputs from the tool result, not shell variables. The ONLY
exception is the two `GIT_INDEX_FILE=.php-dev-loop/snap-index git ...`
snapshot commands above, used verbatim.

## The Task List — decompose before any dispatch

Build `.php-dev-loop/tasks/` before the first implementer dispatch. Each
task gets `task-N-brief.md` — the single source of requirements for that
task; its spec-compliance review is checked against it.

**A plan exists** (the user provided one, or a planning skill such as
superpowers:writing-plans produced one — recognizable by `### Task N:`
sections with Files/steps, typically under `docs/superpowers/plans/`):
consume it task-by-task. Its decomposition IS the task list — do NOT
flatten it into one brief.

- Copy each plan task's full text verbatim into its own
  `task-N-brief.md` — including the plan's Global Constraints section in
  every brief, since every task is implicitly bound by it.
- Never hand a subagent the whole plan file; a task's implementer sees
  only its own brief.
- Scan the plan once for contradictions between tasks or with its Global
  Constraints; batch anything you find into ONE question to the user
  before execution starts.

**No plan** (ticket or ad-hoc ask): derive the task list yourself — this
step is NOT optional and "the request is simple" does not skip it. Split
where a reviewer could approve one task while rejecting its neighbor; fold
setup/scaffolding into the task whose deliverable needs it. A genuinely
small ask is legitimately ONE task — one entry in `tasks/`, same flow.
Each brief you derive must contain:

- Expected behavior as testable statements ("GET /campaigns?archived=true
  returns only rows where archivedAt IS NOT NULL"), edge cases (null,
  empty, invalid input), and what must NOT change
- What the tests must prove
- Exact names/signatures/values where the user gave them; where they
  didn't, explore the code first (semble) and write your chosen convention
  into the brief so the reviewer can hold the implementer to it
- **Interfaces:** what this task consumes from earlier tasks and what
  later tasks rely on — exact signatures. A task's implementer sees only
  its own brief; this block is how neighboring tasks stay consistent.

If the ask is too vague to produce verifiable criteria, ask the user before
dispatching — one batched question, not a guess.

## The Per-Task Loop

Run tasks strictly in order, one implementer in flight at a time. For task
N (per-task files live in `.php-dev-loop/tasks/`):

1. **Dispatch implementer** using `.php-dev-loop/skill/implementer-prompt.md`.
   Give it: one line of scene-setting (where task N fits), the brief path
   (`tasks/task-N-brief.md`), the report path (`tasks/task-N-report.md`),
   the repo path, the PHPUnit container command. Plus interfaces/decisions
   from already-approved tasks that the brief cannot know — nothing else;
   no history of prior tasks.
2. **Handle status:** `DONE` → continue. `DONE_WITH_CONCERNS` → read
   concerns; correctness/scope concerns get resolved before review.
   `NEEDS_CONTEXT` → provide it, re-dispatch. `BLOCKED` → more context, a
   more capable model, a smaller task, or escalate to the user. Never
   re-dispatch unchanged.
3. **Package the task diff** (never into your own context) against the
   PREVIOUS baseline, so the reviewer sees only what task N changed:
   ```bash
   git add -N .   # make new files visible to diff
   git diff --stat "$(cat .php-dev-loop/baseline-<N-1>)" > .php-dev-loop/tasks/task-N-diff.txt
   git diff -U10 "$(cat .php-dev-loop/baseline-<N-1>)" >> .php-dev-loop/tasks/task-N-diff.txt
   ```
   Run these as separate git-prefixed commands, not one `{ }` compound.
4. **Dispatch the iteration reviewer** using
   `.php-dev-loop/skill/reviewer-prompt.md` with the task's brief, report,
   and diff paths.
5. **Verdict:** Critical/Important findings → dispatch ONE fix subagent
   with the complete findings list (fixer re-runs covering tests, appends
   to the task's report) → regenerate the task diff → re-review. Repeat
   until approved. Minor findings: record them in the ledger — they go to
   the final review, never silently dropped.
6. **Snapshot the new baseline:** re-run the two snapshot commands from
   Setup step 3 exactly, write the printed tree SHA to
   `.php-dev-loop/baseline-N`, and append to the ledger:
   `Task N: approved (baseline <first 7 chars>; minors: <list or none>)`.
   This freezes task N's approved state — task N+1's reviewer diffs
   against it and never re-reads approved work.
7. **Next task** — or, after the last one, the final gate.

## Final Gate

Dispatch the final reviewer (`fable`) once, after every task is approved:

1. Package the WHOLE diff — baseline-0 to worktree — into
   `.php-dev-loop/final-diff.txt` (same three commands, baseline-0).
2. Dispatch with `.php-dev-loop/skill/reviewer-prompt.md`: the full diff,
   all task briefs, all reports, plus the ledger's accumulated Minor
   findings to triage. This gate catches what per-task review cannot:
   cross-task seams, drift between briefs, cumulative design smells.
3. Critical/Important → ONE fix dispatch with the complete findings list —
   never one fixer per finding → regenerate final diff → re-review by the
   final reviewer.

## Clean up and report

Remove the scratch space with `git clean -fdX .php-dev-loop/` — `-X`
deletes only git-ignored files and the path scopes it to the scratch dir.
Then report to the user: tasks completed, files changed, test evidence,
findings fixed per task, Minor items left, review iteration counts. State
that nothing is committed and staging awaits their go-ahead.

## Tests

The implementer (and every fixer) runs the tests and reports output —
reviewers do not re-run suites. Every implementer/fixer dispatch carries
the project's test command, resolved by this rule:

- If the user, plan, or brief states a test command → use it verbatim.
- Otherwise → you (the controller, in the current session) decide it from
  what's available: the project's CLAUDE.md, composer.json scripts,
  Makefile, CI config, or an established container setup. Write your
  chosen command into the dispatch — the implementer never guesses it.

Focused filter while iterating; the relevant suite once before reporting
DONE.

## Red Flags — STOP

- **Any `git commit`, `git stash`, or `git add` (except `add -N` and the
  two `GIT_INDEX_FILE=.php-dev-loop/...` snapshot commands) by you or any
  subagent.** Same for any `git clean` other than exactly
  `git clean -fdX .php-dev-loop/` — broader forms destroy the user's
  untracked files. "The skill implies permission" — it does not. "The
  reviewer needs a commit to diff" — it diffs baselines. Approved code is
  still uncommitted code.
- Flattening a provided plan into one brief, or executing a multi-part
  ticket as a single task ("decomposing is overhead") — decomposition is
  where the quality comes from.
- Starting task N+1 before task N's review approved, or running two
  implementers in parallel.
- Skipping the baseline snapshot after an approval — the next reviewer
  would re-see (and re-litigate) approved work.
- Implementing or fixing code yourself in the controller session ("it's a
  one-line fix") — dispatch a fixer; controller context stays clean.
- Skipping re-review after a fix ("the fix was trivial") — every fix diff
  gets reviewed.
- Pasting diff/report/plan contents into a dispatch prompt — pass file
  paths.
- Telling a reviewer what not to flag, or pre-rating severity.
- Skipping the final-reviewer pass because every iteration review
  approved — they are different gates.
- Dispatching a role without its role file ("the model knows how to
  review") — every dispatch starts by reading its `agents/*.md` role.
- Accepting a DONE report with no test command + output in it.
- Re-dispatching a task the ledger already marks approved — after
  compaction or resume, the ledger and the baselines are the truth.
