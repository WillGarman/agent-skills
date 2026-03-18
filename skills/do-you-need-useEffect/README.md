# Do You Need useEffect?

Strict useEffect discipline for React. Treats direct `useEffect` as effectively banned. Every `useEffect` must justify its existence, and when one is genuinely needed, it lives in a named custom hook.

## Sources

- [React: You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
- [Arcjet: Why we banned React's useEffect](https://x.com/ArcjetHQ/status/1901668174789853267) (1.7M views, 4.7K likes, Mar 2026)
- Real production infinite-loop bugs from unstable callbacks in dep arrays

## What It Does

When active, the agent:

1. Refuses to write naked `useEffect` in component bodies
2. Suggests alternatives first (derived state, event handlers, key props, useMemo, useSyncExternalStore)
3. Flags existing `useEffect` during review with specific mechanical fixes
4. Wraps legitimate effects in custom hooks with descriptive names
5. Detects infinite loop patterns (unstable callbacks in dep arrays)

## Triggers

- Writing or reviewing React components with `useEffect`
- `useState` being used for derived values
- Data fetching, subscriptions, state synchronization
- Debugging re-renders, infinite loops, race conditions
- Refactoring useEffect-heavy components

## Files

- `SKILL.md` - Decision tree, rules, legitimate uses, architecture, quick reference, review checklist
- `anti-patterns.md` - 11 cataloged anti-patterns with before/after code
