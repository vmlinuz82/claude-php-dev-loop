---
name: php-dev-loop
description: Use when implementing a feature, bugfix, or Jira ticket in any PHP project (modern Symfony or legacy alike) that should go through a subagent write → review → rework loop before being shown to the user
---

# PHP Dev Loop

Execute one task by dispatching a fresh implementer subagent, reviewing the
working-tree diff with a reviewer subagent, and looping fix → re-review until
approved. Finish with one adversarial final review. **Never commit** — the
loop ends at an approved, uncommitted diff the user commits themselves.

**Core principle:** implementer and reviewers never share context. You (the
controller) pass artifacts as files, not pasted text, and never let an agent
review its own work.

**Continuous execution:** do not check in with the user between iterations.
Stop only for: BLOCKED you cannot resolve, genuine ambiguity, or the loop
completing.

## Roles and Models

The skill is self-contained: every subagent is dispatched as
`general-purpose` and adopts a role bundled in this skill's `agents/`
directory — no pre-installed agents required. Every dispatch prompt MUST
begin with:

> Read `<SKILL_DIR>/agents/<role file>` and adopt that role completely
> before doing anything else. (`<SKILL_DIR>` = the directory containing
> this SKILL.md.)

| Role | Role file | Model | When |
|------|-----------|-------|------|
| Implementer | `agents/php-implementer.md` | `opus` for multi-file work or design judgment; `fable` for architectural decisions or high-risk changes (payments, auth, data migrations); `sonnet` for mechanical 1-2 file tasks with a complete spec | Initial implementation |
| Fixer | `agents/php-implementer.md` | `sonnet` (`haiku` for a one-line mechanical fix) | Review findings |
| Iteration reviewer | `agents/iteration-reviewer.md` | `sonnet`; `opus` when the diff is large, subtle, or security-sensitive | After every implement/fix dispatch |
| Final reviewer | `agents/final-reviewer.md` | `fable` — always the most capable | Once, after the iteration reviewer approves |

All roles share `agents/coding-standards.md` — edit standards there, once.

**Always set `model:` explicitly on every dispatch.** An omitted model
inherits the session's model — often the most expensive — which silently
defeats this table. Escalation ladder is `sonnet → opus → fable`, never
downgrade: a fixer facing a design-level finding gets re-dispatched on
`opus` (or `fable` if the finding is architectural); a model that reported
BLOCKED gets re-dispatched on the next tier up, never retried unchanged.
Turn count beats token price — `haiku` only for pure transcription-level
work; `sonnet` is the floor for anything requiring judgment, including all
reviews.

## Setup (before first dispatch)

Work from the repository the task targets.

```bash
SCRATCH="/tmp/php-dev-loop/$(basename "$(git rev-parse --show-toplevel)")-$(git branch --show-current)"
mkdir -p "$SCRATCH"
BASELINE=$(git stash create); BASELINE=${BASELINE:-$(git rev-parse HEAD)}
echo "$BASELINE" > "$SCRATCH/baseline"
```

`git stash create` snapshots a dirty worktree into an unreferenced commit
WITHOUT modifying anything; empty output means the tree is clean → use HEAD.
Pre-existing changes are then invisible to the reviewer — it sees only what
the loop produced.

## The Brief — always, ticket or not

Write the task brief to `$SCRATCH/brief.md` before any dispatch. It is the
single source of requirements — spec-compliance review is checked against it,
so no brief = unreviewable code.

**With a Jira ticket:** paste the ticket text and acceptance criteria
verbatim, plus exact values/signatures and constraints.

**Without a ticket (ad-hoc ask):** derive the brief yourself — this step is
NOT optional and "the request is simple" does not skip it. Convert the ask
into verifiable acceptance criteria:

- Expected behavior as testable statements ("GET /campaigns?archived=true
  returns only rows where archivedAt IS NOT NULL"), including edge cases
  (null, empty, invalid input) and what must NOT change
- What the tests must prove
- Exact names/signatures/values where the user gave them; where they didn't,
  explore the code first (semble) and write your chosen convention into the
  brief so the reviewer can hold the implementer to it

If the ask is too vague to produce verifiable criteria, ask the user before
dispatching — one batched question, not a guess.

## The Loop

1. **Dispatch implementer** using [implementer-prompt.md](implementer-prompt.md).
   Give it: one line of scene-setting, the brief path, the report path
   (`$SCRATCH/report.md`), the repo path, the PHPUnit container command.
2. **Handle status:** `DONE` → continue. `DONE_WITH_CONCERNS` → read concerns;
   correctness/scope concerns get resolved before review. `NEEDS_CONTEXT` →
   provide it, re-dispatch. `BLOCKED` → more context, a more capable model, a
   smaller task, or escalate to the user. Never re-dispatch unchanged.
3. **Package the diff** (never into your own context):
   ```bash
   git add -N .   # make new files visible to diff
   { git status --porcelain; echo; git diff --stat "$(cat "$SCRATCH/baseline")"; echo; git diff -U10 "$(cat "$SCRATCH/baseline")"; } > "$SCRATCH/diff.txt"
   ```
4. **Dispatch the iteration reviewer** using [reviewer-prompt.md](reviewer-prompt.md)
   with the brief, report, and diff paths.
5. **Verdict:** Critical/Important findings → dispatch ONE fix subagent with
   the complete findings list (fixer re-runs covering tests, appends to
   report) → regenerate diff → re-review. Repeat until approved. Minor
   findings: record, hand to the final review — do not silently drop them.
6. **Final gate:** dispatch the final reviewer with the same three files plus
   the Minor-findings list, using [reviewer-prompt.md](reviewer-prompt.md).
   Critical/Important → one fix dispatch → re-review by the final reviewer.
7. **Report to the user:** what was built, files changed, test evidence,
   findings fixed, Minor items left, review iteration count. State that
   nothing is committed and staging awaits their go-ahead.

## Tests

The implementer (and every fixer) runs the tests and reports output —
reviewers do not re-run suites. Every implementer/fixer dispatch carries the
project's test command, resolved by this rule:

- If the user or brief states a test command → use that command verbatim.
- Otherwise → you (the controller, in the current session) decide it from
  what's available: the project's CLAUDE.md, composer.json scripts, Makefile,
  CI config, or an established container setup. Write your chosen command
  into the dispatch — the implementer never guesses it.

Run a focused subset while iterating; the relevant suite once before DONE.

Focused filter while iterating; relevant suite once before reporting DONE.

## Red Flags — STOP

- **Any `git commit`, `git stash push/pop`, or `git add` (except `add -N`) by
  you or any subagent.** "The skill implies permission" — it does not. "The
  reviewer needs a commit to diff" — it diffs the baseline. Approved code is
  still uncommitted code.
- Implementing or fixing code yourself in the controller session ("it's a
  one-line fix") — dispatch a fixer; controller context stays clean.
- Skipping re-review after a fix ("the fix was trivial") — every fix diff gets
  reviewed.
- Pasting diff/report contents into a dispatch prompt — pass file paths.
- Telling a reviewer what not to flag, or pre-rating severity.
- Skipping the final-reviewer pass because the iteration reviewer approved —
  they are different gates.
- Dispatching a role without its role file ("the model knows how to review")
  — every dispatch starts by reading its `agents/*.md` role.
- Accepting a DONE report with no test command + output in it.
