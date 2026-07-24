# Coding Standards (shared by all php-dev-loop roles)

Violations of these are review findings. When writing code, follow them; when
reviewing, enforce them.

## Applicability — determine the project profile FIRST

Establish from `composer.json` (`require.php` / `platform.php`, framework
packages) and the surrounding code what the project is. Then:

- Sections whose heading carries a condition apply only when it holds.
- **Project static-analysis config is binding.** If the project ships
  PHPStan, PHP_CodeSniffer, or PHPMD configuration (rulesets, levels,
  baselines), those rules are the project's codified conventions: write
  code that passes them, and treat reported violations as review findings.
  Where a project ruleset conflicts with the generic Style section below,
  the project ruleset wins. Never edit tool configs or baseline files to
  silence a violation.
- Control Flow, SOLID, and Security apply to EVERY project — they are
  REQUIRED, never optional. If a rule names a specific tool the project
  doesn't have (DQL/QueryBuilder, serialization groups), apply the closest
  equivalent it does have (PDO prepared statements, framework escaping,
  explicit field whitelisting) — the rule itself is never optional.
- The legacy-consistency clause below covers STYLE and LANGUAGE FEATURES
  only. It is never a license to violate SOLID in new code: no new god
  classes, tight coupling, or untestable designs, regardless of how the
  surrounding legacy code is built. New units are written SOLID; only their
  surface style matches the codebase.
- Language features apply only where the project's PHP version supports
  them — never suggest a feature newer than the declared version.
- **Legacy codebases** (old PHP versions, framework-less or outdated
  stacks): consistency with the existing style wins over modernization.
  Don't introduce `strict_types`, type declarations, or modern idioms into
  files that don't use them; flag an existing legacy pattern only when the
  change at hand makes it a real risk.

## Language — modern PHP
Target the project's own PHP version — read `composer.json` `require.php`
(don't assume a version; projects upgrade). Use the modern features that
version offers:
- `declare(strict_types=1);` at the top of every PHP file
- Enums over class constants for finite value sets
- `readonly` properties (and classes) where state must not change after construction
- Constructor property promotion; named arguments where they aid readability
- `match` over `switch` where appropriate; first-class callable syntax
- Never `mixed` unless truly unavoidable — explicit types everywhere

## Control Flow
- Never use `else` — early returns and guard clauses instead
- Maximum nesting depth 2; prefer 0–1 via early returns or method extraction
- Methods short (~20 lines); extract when they grow

## Style — PSR-12 (in legacy codebases, match the dominant existing style instead)
- 4-space indent; braces same line for control structures, next line for classes/methods
- Visibility on all properties/methods; alphabetical, grouped use statements
- No trailing whitespace; single blank line at EOF

## Symfony (only if the project uses Symfony — composer.json requires symfony/*)
- Autowiring/autoconfiguration; constructor injection only (no setter injection, no service locator)
- PHP attributes for routing, validation, serialization groups, security
- Voters for complex authorization, `#[IsGranted]` for simple cases
- Events/subscribers over direct coupling; Messenger for async
- Validate ALL input via the Validator component or forms — never trust raw input

## API Platform (only if composer.json requires api-platform/*)
- Attribute-based resource config; custom State Providers/Processors over controller overrides
- DTOs when API shape differs from entity shape; explicit serialization groups
- Built-in filtering/pagination/validation; custom operations over custom controllers

## SOLID
- Single responsibility per class; split god classes
- Open/closed via interfaces and strategies; composition over inheritance
- Depend on abstractions; inject interfaces where multiple implementations or mocking are plausible

## Security — be paranoid (OWASP Top 10)
- SQL injection: parameterized queries via DQL/QueryBuilder only — never concatenate input
- Mass assignment: explicit DTOs/serialization groups — never bind raw request data to entities
- XSS/data leakage: escape output, tight serialization groups
- AuthN/AuthZ on every endpoint; CSRF where applicable; no secrets/PII in logs or code
- Path traversal, insecure deserialization, `eval`/`exec`-family misuse: flag on sight

## Doctrine (only if the project uses Doctrine ORM)
- Watch for N+1 queries; correct fetch modes; indexes on filtered/sorted columns
- Transactions for multi-write integrity; deliberate cascade configuration

## Tests (follow the project's own test framework; PHPUnit-specific rules apply where PHPUnit is used. If the project has no test suite, say so in the report — don't build test infrastructure unasked)
- One test method per tested method; multiple cases via a data provider returning `\Generator`
- Assert exceptions with `$this->expectException(...)` BEFORE the call — never try/catch in tests
- Follow sibling test files' naming, base classes, and mocking patterns exactly
- Tests must assert real behavior, not mock choreography
