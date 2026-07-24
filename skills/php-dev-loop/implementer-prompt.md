# Implementer Dispatch Template

For the first implementation dispatch and for fix dispatches (adapt the Job
section for fixes). Agent type is always `general-purpose`; the role comes
from the bundled role file.

```
Agent (general-purpose):
  description: "Implement: [short task name]"
  model: [OMIT — the dispatch inherits the session model, which is the
         rule for implementers and fixers. Set it only when SKILL.md's
         Roles and Models section calls for a different tier: an
         escalation, or a model the user named]
  prompt: |
    Read [SCRATCH]/skill/agents/php-implementer.md and adopt that role
    completely before doing anything else — including the coding-standards
    file it points to.

    You are implementing one task in [repo path].

    ## Task

    Read your brief first: [SCRATCH]/tasks/task-[N]-brief.md
    It is your requirements — exact values, signatures, and acceptance
    criteria in it are binding, verbatim.

    ## Context

    [One or two lines: where this task fits, relevant existing
    services/patterns, plus interfaces and decisions from already-approved
    tasks that the brief cannot know — never a history of prior tasks]

    ## Before You Begin

    If requirements, approach, or assumptions are unclear — ask now, before
    writing code. Don't guess.

    ## Your Job

    1. Implement exactly what the brief specifies — nothing extra
    2. Write/update tests following the conventions of sibling test files
       (read 3 existing tests in the same directory first)
    3. Run tests with: [TEST COMMAND — controller fills this in per
       SKILL.md Tests and Static Analysis; the implementer never chooses
       its own runner]
       Focused subset while iterating; the relevant suite once before reporting.
    4. Run static analysis on the files you changed with:
       [STATIC ANALYSIS COMMANDS — controller fills these in per SKILL.md
       Tests and Static Analysis; DELETE this step if Setup found no
       configured tools]
       The project's rulesets and baselines are binding — fix violations;
       never edit tool configs or baseline files to silence them.
    5. Self-review: completeness vs brief, edge cases, coding standards,
       tests assert real behavior
    6. Report back

    ## Hard Rules

    - NEVER run `git commit`, `git add`, `git stash`, or any command that
      mutates git state. Your output is uncommitted working-tree changes only.
    - Touch only what the task requires. A changed line that doesn't trace
      to the brief shouldn't exist.
    - If the task needs architectural decisions with multiple valid options,
      or you can't gain clarity from the code — STOP and escalate. Bad work
      is worse than no work.

    ## Report

    Write your full report to [SCRATCH]/tasks/task-[N]-report.md:
    - What you implemented (or attempted)
    - Test command run + relevant output
    - Static analysis command(s) run + output, if the project has them
      configured
    - Files changed
    - Self-review findings
    - Concerns

    Then reply with ONLY (under 10 lines):
    - Status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT
    - One-line test summary (e.g. "12/12 passing")
    - Concerns, if any

    If BLOCKED or NEEDS_CONTEXT, put the specifics in the reply itself.
    Never silently produce work you're unsure about.
```

**Fix dispatch differences:** replace "Your Job" step 1 with the complete
findings list from the reviewer (all Critical + Important, verbatim, with
file:line). The fixer re-runs the tests covering its changes plus the
static-analysis commands over the files it touched, and APPENDS a fix
section to the same report file. Before re-dispatching the reviewer,
confirm the fix report names the covering tests, the commands, and the
output.
