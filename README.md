# PHP Dev Loop

A [Claude Code](https://claude.com/claude-code) skill that runs every PHP/Symfony
task through a **write → review → rework loop** before you ever see the code:

```
brief → implementer subagent → diff → iteration reviewer
            ▲                              │
            └────── fix subagent ◄── Critical/Important findings
                                           │ approved
                                           ▼
                              final adversarial reviewer
                                           │ approved
                                           ▼
                        approved, UNCOMMITTED diff + report
```

The point: the model that writes the code never reviews it. Implementer and
reviewers run as **fresh subagents with isolated contexts**, hand artifacts to
each other as files (brief, report, diff), and loop until two independent
review gates approve. You get working, twice-reviewed, test-backed code — and
**nothing is ever committed**; the loop ends at an uncommitted diff you commit
yourself.

## Features

- **Two staged review gates** — a strict diff reviewer on every iteration, plus
  one adversarial "assume the author missed something" reviewer as the final gate
- **Works with or without a ticket** — given a Jira ticket it reviews against
  its acceptance criteria; given a vague ask it first derives verifiable
  acceptance criteria itself, so even ad-hoc code gets a real spec review
- **Model tiering** — cheap models for mechanical work, top models
  (Opus/Fable) for design and the final gate; every dispatch pins its model
  explicitly
- **Self-contained** — the implementer/reviewer roles and the PHP coding
  standards are bundled markdown files; no custom agents to install
- **Never touches git state** — no commits, no staging, no stash mutations, by
  you or any subagent; enforced as a hard rule on every role

## Requirements

- Claude Code (this skill is Claude-only by design — it relies on Claude
  Code's subagent dispatch and skill auto-triggering)
- A PHP project in a git repository, with PHPUnit tests

## Install

### As a plugin (recommended)

```
/plugin marketplace add vmlinuz82/claude-php-dev-loop
/plugin install php-dev-loop@vmlinuz82-plugins
```

Update later with:

```
/plugin marketplace update vmlinuz82-plugins
/plugin update php-dev-loop
```

### Manual copy

```bash
git clone https://github.com/vmlinuz82/claude-php-dev-loop /tmp/claude-php-dev-loop
cp -r /tmp/claude-php-dev-loop/skills/php-dev-loop ~/.claude/skills/
```

> If you have both the plugin and a manual copy in `~/.claude/skills/`, the
> local copy wins. Pick one.

## Usage

Invoke explicitly:

```
/php-dev-loop implement EM-1234: <paste ticket text>
```

or just ask for PHP work that deserves review — the skill triggers on its own:

```
add retry with backoff to the webhook sender, make sure it's solid quality
```

Claude then writes a brief with verifiable acceptance criteria, runs the loop,
and reports back: what was built, files changed, test evidence, findings fixed
per review round, and anything minor left over. You review the diff and commit
when satisfied.

## What's inside

```
skills/php-dev-loop/
├── SKILL.md                  # controller loop, roles & models, red flags
├── implementer-prompt.md     # dispatch template: implement + fix
├── reviewer-prompt.md        # dispatch template: iteration + final review
└── agents/
    ├── coding-standards.md   # ALL standards, single source — edit here
    ├── php-implementer.md    # implementer role
    ├── iteration-reviewer.md # strict per-iteration diff reviewer
    └── final-reviewer.md     # adversarial final gate
```

## Customizing

- **Coding standards** (strict types, no `else`, max nesting, PSR-12, SOLID,
  OWASP, test conventions) live in one file:
  `skills/php-dev-loop/agents/coding-standards.md`. Edit it there — all three
  roles read it.
- **PHP version** is never pinned — roles target whatever the project's
  `composer.json` declares.
- **Test command** is decided per session: whatever you or the project
  (composer scripts, Makefile, CI config, project CLAUDE.md) defines.
- **Models** are configured in the table in `SKILL.md` if your cost/quality
  trade-offs differ.

## License

[MIT](LICENSE)
