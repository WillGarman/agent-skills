# Make It Readable

Code readability cleanup pass. Run on changed code before every PR.

```
npx skills add WillGarman/agent-skills --skill make-it-readable
```

## Source

- [@gabriel1: make code skimmable, run a separate cleanup prompt before every pr](https://x.com/gabriel1/status/1902049241707389100) (54K views, 745 likes, Mar 2026)

## What It Does

When active, the agent reviews changed code and applies these principles:

1. Makes code skimmable — obvious at a glance
2. Minimizes possible states (fewer args, narrower types)
3. Replaces optional fields with discriminated unions
4. Adds exhaustive handling with `assertNever` for unknown variants
5. Removes defensive code in favor of trusting types
6. Adds boundary asserts instead of silent defaults
7. Deletes dead code, unused imports, commented blocks
8. Condenses verbose code into fewer lines
9. Replaces clever code with obvious code
10. Inlines over-extracted single-use helpers
11. Flattens nested conditionals with early returns
12. Replaces try-catches with asserts for expected-to-exist values
13. Removes speculative overrides, keeps argument count low
14. Makes actually-required arguments required, not optional

## Triggers

- Pre-PR code cleanup
- Reviewing diffs for readability
- Refactoring over-engineered or defensive code
- Code that feels hard to follow or has too many optional parameters

## Files

- `SKILL.md` - All 14 rules with before/after code examples, quick reference table, review checklist
