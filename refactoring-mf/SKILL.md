---
name: refactoring-mf
description: Refactoring patterns catalog based on Martin Fowler's book, adapted for Rust. Use when refactoring code, identifying code smells, choosing refactoring strategies, or when the user asks about improving code structure.
---

# Refactoring Patterns Catalog

A complete reference of refactoring patterns from Martin Fowler's "Refactoring: Improving the Design of Existing Code" (2nd Edition), adapted for Rust.

Use this when you need to:
- Identify code smells and choose appropriate refactorings
- Apply specific refactoring mechanics step-by-step
- Follow Rust-specific refactoring patterns (ownership, traits, error handling)
- Reference the smell-to-refactoring mapping

## Key Principles

- **Two Hats:** Never add functionality and refactor simultaneously
- **Rule of Three:** First time just do it, second time wince, third time refactor
- **Small Steps:** Compile and test after each change
- **Name After Intent:** Functions named after what they do, not how

## Resources

- For the full catalog with all patterns, mechanics, and Rust examples, see [reference.md](references/reference.md)