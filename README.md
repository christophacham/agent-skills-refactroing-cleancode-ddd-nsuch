# Agent Skills — Refactoring, Clean Code & DDD

Agent skills for structured software development with an emphasis on clean architecture, domain-driven design, and systematic refactoring.

## Repository: [christophacham/clean-architecture-ddd-example](https://github.com/christophacham/clean-architecture-ddd-example)

A complete, runnable Rust example accompanying the `clean-architecture-ddd` skill.

### How the Skill and Repository Relate

| What | Skill (`clean-architecture-ddd/`) | Repo (`clean-architecture/`) |
|------|-----------------------------------|-------------------------------|
| **Purpose** | Theory, patterns, instructions for the agent | Concrete, compilable, runnable implementation |
| **Content** | SKILL.md (guide) + references/reference.md (deep dive) | `src/` (code) + `tests/` + `README.md` |
| **You use it when** | *Writing new code* — agent follows the recipe (domain → app → infra) | *Learning or verifying* — run `./run.sh demo` to see it work, `./run.sh test` to confirm |
| **What it gives you** | Mental model, cookbook, anti-patterns, Rust-specific guidance | 24 passing tests, code you can step through, edit, and extend |
| **Coverage** | All patterns (aggregate, value objects, events, ports, adapters, UI as detail) | Same patterns, implemented and tested |

**The skill references the repo** — code snippets in SKILL.md and reference.md are taken directly from `src/`. The repo's `README.md` links back to the skill for deeper learning.

**The repo is the truth** — if you want to understand a pattern, run the tests. If you want to apply it, follow the skill.

---

## Skills

### `clean-architecture-ddd`
Clean Architecture + Domain-Driven Design patterns in Rust. Use when structuring new projects, separating concerns, designing domain models, or when working with layered architecture, dependency inversion, aggregates, value objects, or domain events.

**Location:** `/home/kevin/.agents/skills/clean-architecture-ddd/`  
**Example project:** `/home/kevin/source/clean-architecture/`

### `refactoring-mf`
Refactoring patterns catalog based on Martin Fowler's book, adapted for Rust. Use when refactoring code, identifying code smells, choosing refactoring strategies, or improving code structure.

**Location:** `/home/kevin/.agents/skills/refactoring-mf/`
