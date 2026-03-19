---
name: do-you-need-useEffect
description: Strict useEffect discipline for React. Treats direct useEffect as effectively banned, requiring developers to exhaust all React primitives (derived state, event handlers, key props, useMemo, useSyncExternalStore) before reaching for an effect. When an effect is genuinely needed, it must live in a named custom hook. Use when writing or reviewing any React component, refactoring useEffect-heavy code, debugging infinite loops or unnecessary re-renders, or when a component has more than zero useEffect calls.
---

# Do You Need useEffect? (No, You Probably Don't)

Direct `useEffect` is banned until proven necessary. Every `useEffect` in a component body is a claim that React's rendering model isn't enough. That claim is wrong ~90% of the time.

## The Rule

1. **Never reach for useEffect first.** Exhaust every alternative below.
2. **No naked useEffect in component bodies.** If you genuinely need one, wrap it in a custom hook with a descriptive name (`useMapInit`, `useWebSocket`, `useDocumentTitle`).
3. **Every useEffect is guilty until proven innocent.** During review, each one must justify its existence.

A naked `useEffect` in a component means one of four things:
1. You're computing derived state (just calculate it)
2. You're responding to a user event (use the event handler)
3. You're syncing with an external system (wrap in a custom hook)
4. You wrote an infinite loop waiting to happen

## Decision Tree

```
Before writing useEffect, ask:
│
├─ Can I calculate this from existing props/state?
│  ├─ AND the user can never change it independently → DERIVE INLINE or useMemo
│  └─ BUT the user can change it after initial load → See "Seed Once" below
│
├─ Am I seeding state once from async data, then letting the user own it?
│  └─ setState DURING RENDER with a ref guard (no useEffect needed)
│     See "Seed Once from Async Data" section below
│
├─ Am I responding to a user action (click, submit, type, drag)?
│  └─ EVENT HANDLER. No exceptions.
│
├─ Do I need to reset ALL state when a prop changes?
│  └─ KEY PROP: <Profile key={userId} />
│
├─ Do I need to adjust SOME state when a prop/state changes (but keep the rest)?
│  └─ setState DURING RENDER with a "previous value" comparison
│     Track the previous value, compare, update if changed. See below.
│
├─ Am I subscribing to an external store / browser API?
│  └─ useSyncExternalStore
│
├─ Am I notifying a parent about state changes?
│  └─ Call the callback IN THE EVENT HANDLER
│     Or better: make the component fully controlled
│
├─ Am I passing data up to a parent?
│  └─ LIFT the data fetching to the parent, pass down as props
│
├─ Am I initializing a third-party library on mount?
│  └─ CUSTOM HOOK: useMapboxInit(ref, options)
│
└─ None of the above, genuinely syncing with external system?
   └─ CUSTOM HOOK. Never naked useEffect.
```

## The 5 Legitimate Effect Uses

All must live in custom hooks, never as bare `useEffect` in a component.

### 1. Mount-only side effects

Analytics, one-time initialization. Use a named hook:

```tsx
function useMountEffect(fn: () => void | (() => void)) {
  // eslint-disable-next-line react-hooks/exhaustive-deps
  useEffect(fn, []);
}

function Analytics({ pageId }: { pageId: string }) {
  useMountEffect(() => { track('page_view', { pageId }); });
}
```

### 2. External system synchronization

Third-party libraries, imperative APIs:

```tsx
function useMapInit(ref: RefObject<HTMLDivElement>, center: LatLng) {
  useEffect(() => {
    const map = new Map(ref.current, { center });
    return () => map.destroy();
  }, [ref, center]);
}
```

### 3. DOM measurement / observers

ResizeObserver, IntersectionObserver, MutationObserver:

```tsx
function useResizeObserver(ref: RefObject<HTMLElement>, onResize: (entry: ResizeObserverEntry) => void) {
  useEffect(() => {
    if (!ref.current) return;
    const observer = new ResizeObserver(([entry]) => onResize(entry));
    observer.observe(ref.current);
    return () => observer.disconnect();
  }, [ref, onResize]);
}
```

### 4. Subscriptions

Prefer `useSyncExternalStore`. Fall back to custom hook only when the API doesn't fit the store model:

```tsx
// Prefer this
function useOnlineStatus() {
  return useSyncExternalStore(
    (cb) => {
      window.addEventListener('online', cb);
      window.addEventListener('offline', cb);
      return () => { window.removeEventListener('online', cb); window.removeEventListener('offline', cb); };
    },
    () => navigator.onLine,
    () => true
  );
}
```

### 5. Data fetching

Use your framework's mechanism (Next.js server components, loader functions) or a library (React Query, SWR, Convex `useQuery`). Raw fetch-in-useEffect is the last resort, and it MUST have cleanup:

```tsx
// Last resort, must have cleanup
function useData<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  useEffect(() => {
    let ignore = false;
    fetch(url).then(r => r.json()).then(json => { if (!ignore) setData(json); });
    return () => { ignore = true; };
  }, [url]);
  return data;
}
```

## Seed Once from Async Data (The Tricky One)

This pattern is deceptive because it *looks* like derived state but isn't. The tell: after the initial seed, the user can change the value independently. That means it's not derived, it's **initialized from async data**.

**The trap**: Making it fully derived kills interactivity. Making it a `useEffect` adds unnecessary timing complexity. Neither is right.

**The fix**: React supports calling `setState` during render. When React sees it, it bails out of the current render and immediately re-renders with the new value. Combined with a ref guard, this seeds exactly once with zero effect timing issues.

```tsx
// WRONG: fully derived — user can never change the selection
const selectedIds = useMemo(() => {
  const match = options?.find(o => o.type === defaultType);
  return match ? new Set([match.id]) : new Set();
}, [options, defaultType]);
// Combobox changes are ignored because the value is always recomputed ^

// WRONG: useEffect to seed — works but adds unnecessary effect timing
const [selectedIds, setSelectedIds] = useState<Set<string>>(new Set());
useEffect(() => {
  if (!hasSeeded.current && options && defaultType) {
    const match = options.find(o => o.type === defaultType);
    if (match) { setSelectedIds(new Set([match.id])); hasSeeded.current = true; }
  }
}, [options, defaultType]);

// RIGHT: setState during render with ref guard — no effect needed
const [selectedIds, setSelectedIds] = useState<Set<string>>(new Set());
const hasSeededRef = useRef(false);

if (!hasSeededRef.current && options && defaultType) {
  const match = options.find(o => o.type === defaultType);
  if (match) {
    setSelectedIds(new Set([match.id]));
    hasSeededRef.current = true;
  }
}
// After seeding, user changes via setSelectedIds work normally
```

**How to recognize this pattern**:
- State starts empty/default
- Async data arrives (query, fetch, context)
- State should be initialized from that data **once**
- After that, the user owns it (combobox, form, toggle, etc.)

**The key question to ask**: "After the initial value is set, can the user change it independently?" If yes, it's seed-once, not derived state. Use `setState` during render with a ref guard.

**Why not useEffect?** React's `setState` during render is synchronous, the component re-renders immediately with the seeded value. A `useEffect` runs *after* paint, so the user sees a flash of the empty/default state first. The render-time approach also makes the data flow obvious: you can read top-to-bottom and see exactly when and why the seed happens.

## Adjust Some State When a Prop Changes (Not All)

Distinct from the key prop pattern. Use `key` when you want to blow away ALL state and start fresh. Use setState-during-render when you want to adjust ONE piece of state while preserving everything else.

**The pattern**: Track the previous prop value, compare during render, call setState if changed.

```tsx
// Nav item: auto-expand when route becomes active, but preserve user's manual collapse
function NavItem({ active, children }: { active: boolean; children: ReactNode }) {
  const [open, setOpen] = useState(active);
  const [prevActive, setPrevActive] = useState(active);

  // Adjust open when active changes, but don't destroy manual collapse state
  if (active !== prevActive) {
    setPrevActive(active);
    setOpen(active);
  }

  return <Collapsible open={open} onOpenChange={setOpen}>{children}</Collapsible>;
}
```

**Why not key prop?** Key would remount the entire subtree, losing the user's manual collapse and any other local state. We only want to nudge `open` when `active` changes.

**Why not useEffect?** The effect version renders one frame with the stale `open` value (user sees the old state flash), then corrects. The render-time version is synchronous: React bails out and re-renders with both values updated before committing to DOM.

**How to recognize**: "When X prop changes, adjust Y state, but keep Z state intact." If you'd lose Z by using a key prop, this is the right pattern.

## Optimistic UI with Reactive Backends

Common pattern with Convex, Firebase, or any system where server state is pushed reactively. You need instant visual feedback on click, but the server state arrives asynchronously.

The naive approach: mirror server state into local state with a `useEffect` sync. This is anti-pattern #3 (resetting state on prop change).

**The fix**: Nullable override pattern. Override wins when set, server truth wins when override is null. Clear the override when the server catches up (render-time comparison).

```tsx
function ToggleButton({ serverValue, onToggle }: { serverValue: boolean; onToggle: (v: boolean) => Promise<void> }) {
  const [override, setOverride] = useState<boolean | null>(null);
  const lastServerRef = useRef(serverValue);

  // Server caught up, clear the optimistic override
  if (lastServerRef.current !== serverValue) {
    lastServerRef.current = serverValue;
    if (override !== null) setOverride(null);
  }

  const displayValue = override ?? serverValue; // derived, not synced

  const handleClick = async () => {
    const next = !displayValue;
    setOverride(next);
    try {
      await onToggle(next);
    } catch {
      setOverride(null); // revert to server truth on error
    }
  };

  return <Button onClick={handleClick}>{displayValue ? "On" : "Off"}</Button>;
}
```

**Why this works**: `displayPaused` is always derived (override ?? server). The override-clearing is a render-time state adjustment. No useEffect anywhere. The error path reverts gracefully.

## Reduce Before Remove

Sometimes the effect is legitimate (external system sync), but its scope is too broad. Before accepting a useEffect, ask: "Can I move 90% of this into render-time logic and keep the effect only for the irreducible async gap?"

Example: a rich text editor with draft restore. The editor is an imperative API (external system), and its `content` option is only read on mount (the editor ignores prop changes after initialization). Most cases (remount after collapse/expand) can be handled by the initial `content` option. The effect only needs to cover the async edge case where draft data arrives after mount.

```tsx
// BEFORE: effect handles everything
useEffect(() => {
  if (!editor) return;
  if (editor.getText() !== draft) editor.commands.setContent(draft || "");
}, [editor, draft]);

// AFTER: initial option handles 90%, effect handles only the async gap
const editor = useRichTextEditor({ content: draft || "" }); // handles mount/remount

const hasRestoredRef = useRef(false);
useEffect(() => {
  if (!editor || hasRestoredRef.current) return;
  if (draft && editor.isEmpty) {
    editor.commands.setContent(draft);
    hasRestoredRef.current = true;
  }
}, [editor, draft]);
```

When a useEffect passes the legitimacy test, still minimize it. Push everything possible into render-time, initial props, or event handlers. The effect should only cover the irreducible timing gap.

## Anti-Pattern Catalog

See [anti-patterns.md](./anti-patterns.md) for the full catalog with code examples. Summary:

| # | Anti-Pattern | What To Do Instead |
|---|---|---|
| 1 | `useState` + `useEffect` for derived values | Inline calculation: `const x = a + b` |
| 2 | Filtering/transforming data in effect | `useMemo` or inline |
| 3 | Resetting ALL state on prop change | `key` prop |
| 3b | Adjusting SOME state on prop change | setState during render with prev-value comparison |
| 4 | Event-specific logic in effect | Event handler |
| 5 | Effect chains (effects triggering effects) | All state transitions in the event handler |
| 6 | Notifying parent via effect | Call callback in event handler, or go fully controlled |
| 7 | Passing data up to parent via effect | Lift fetch to parent, pass down |
| 8 | Fetching without cleanup | Cleanup flag, AbortController, or use a library |
| 9 | Unstable callback causing infinite loop | `useCallback` on every function prop used in dep arrays |
| 10 | App init logic in effect | Module-level guard or module-level execution |
| 11 | Store object, reset via effect | Store ID, derive object |
| 12 | Seed-once from async data via effect | `setState` during render with ref guard |

## The Infinite Loop Pattern (The Sneakiest Bug)

This deserves special attention because it's the hardest to spot and most common in production.

**The pattern**: A parent passes a callback prop. A child uses it in a `useEffect` dep array. The callback isn't wrapped in `useCallback`, so it's a new reference every render. The effect fires, triggers a state update, parent re-renders, new callback reference, effect fires again. Infinite loop.

```tsx
// BUG: infinite loop
function Parent() {
  // New function every render
  const handleReady = (seekFn: (t: number) => void) => setSeek(() => seekFn);
  return <Player onReady={handleReady} />;
}

function Player({ onReady }: { onReady: (fn: (t: number) => void) => void }) {
  const seekTo = useCallback((time: number) => { /* seek logic */ }, []);
  useEffect(() => {
    onReady(seekTo); // fires every render because onReady changes every render
  }, [onReady, seekTo]);
}
```

```tsx
// FIX: stable callback identity
function Parent() {
  const handleReady = useCallback(
    (seekFn: (t: number) => void) => setSeek(() => seekFn),
    []
  );
  return <Player onReady={handleReady} />;
}
```

**Detection rule**: Any `useEffect` with a function prop in its dep array is a potential infinite loop. The parent MUST wrap that function in `useCallback`. When reviewing code, grep for this pattern.

## Architecture: The Forcing Function

Banning useEffect forces better component tree design:

```
<MainLayout />                       ← mounts once, owns top-level lifecycle
  <AuthProvider key={userId} />      ← key remounts on user change
    <SessionPage key={sessionId} />  ← key remounts on session change
      <MessageHistory />             ← pure render, no effects
      <FileView path={path} />       ← subscribes via custom hook
      <ChatInput />                  ← event handlers only
```

The philosophy:
- **Parents own orchestration and lifecycle boundaries.** They decide when children mount/unmount via keys and conditional rendering.
- **Children assume preconditions are met.** They receive data as props, not fetch it themselves.
- **Each component does one job.** Coordination happens at boundaries, not inside leaf components.

This is Unix philosophy for React: small, composable units with clear interfaces.

When you feel the urge to add `useEffect` to a child, ask:
- Should the **parent** orchestrate this instead?
- Should a **key prop** reset this?
- Should this data **flow down** as props?
- Can this be **derived** from what's already available?

## Choose Your Failure Mode

| | Custom mount hook | Raw useEffect |
|---|---|---|
| **0 calls** | Obvious, caught immediately in dev | — |
| **1 call** | Correct, deterministic | Maybe correct |
| **2+ calls** | Impossible (mount-only) | Race conditions, leaks, infinite loops |
| **Debugging** | Binary: did it mount? | Which dep changed? Why did it re-fire? |

Custom hook failures are **loud and binary** (it ran or it didn't). Raw useEffect failures **degrade gradually**: flaky behavior, performance issues, loops that only surface under specific timing conditions.

## Quick Reference

| You want to... | Not this | This |
|---|---|---|
| Compute from props/state | `useState` + `useEffect` | `const x = a + b` or `useMemo` |
| Filter/sort/transform | `useState` + `useEffect` | `useMemo` or inline |
| Reset ALL state on prop change | `useEffect` + `setState` | `key` prop |
| Adjust SOME state on prop change | `useEffect` + `setState` | setState during render + prev-value ref |
| Optimistic UI with reactive backend | `useEffect` syncing server→local | Nullable override pattern |
| Respond to user action | `useEffect` watching flag | Event handler |
| Notify parent of change | `useEffect` calling callback | Call in event handler |
| Pass data to parent | `useEffect` calling `onFetched` | Lift fetch to parent |
| Subscribe to browser API | Manual `addEventListener` in effect | `useSyncExternalStore` |
| Initialize lib on mount | Naked `useEffect` | Custom hook |
| Fetch data | Naked `useEffect` | Framework / library / custom hook |
| Run once on mount | `useEffect(fn, [])` | `useMountEffect` custom hook |
| Seed state from async data | `useEffect` + ref guard | `setState` during render + ref guard |

## Review Checklist

When you see `useEffect` in code:

- [ ] Is this derived state? → Delete state + effect, calculate inline
- [ ] Is this responding to a user event? → Move to event handler
- [ ] Is this resetting state on prop change? → Use key prop
- [ ] Is this an effect chain? → Consolidate into event handler
- [ ] Is this notifying a parent? → Call in event handler, or make fully controlled
- [ ] Does cleanup exist where needed? → Add it (fetch, subscribe, timer)
- [ ] Is every function in the dep array stable? → Wrap in `useCallback`
- [ ] Is this a naked useEffect? → Extract to a named custom hook
- [ ] Is this seeding state once from async data? → `setState` during render with ref guard
- [ ] Is this genuinely syncing with an external system? → OK, but must be a custom hook
