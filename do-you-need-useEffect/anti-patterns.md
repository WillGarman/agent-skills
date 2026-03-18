# useEffect Anti-Pattern Catalog

Every pattern below is a real bug that ships to production. Each one has a mechanical fix.

## 1. Derived State Syndrome

The #1 useEffect abuse. `useState` + `useEffect` to compute what should be a `const`. Causes an extra render cycle where the UI shows stale data before correcting.

```tsx
// BUG: renders with empty string, then re-renders with correct value
const [fullName, setFullName] = useState('');
useEffect(() => {
  setFullName(firstName + ' ' + lastName);
}, [firstName, lastName]);

// FIX: zero extra renders, always correct
const fullName = firstName + ' ' + lastName;
```

This extends to any transformation: formatting dates, concatenating strings, computing booleans, mapping arrays. If the inputs are props or state, the output is derived state. Just compute it.

```tsx
// More examples of derived state that should NEVER be in useEffect
const isValid = email.includes('@') && password.length >= 8;
const totalPrice = items.reduce((sum, item) => sum + item.price, 0);
const sortedUsers = [...users].sort((a, b) => a.name.localeCompare(b.name));
const activeCount = todos.filter(t => !t.completed).length;
```

---

## 2. Filtering / Transforming in Effect

A specific case of derived state that's extremely common with lists.

```tsx
// BUG: extra render with stale filtered list, then corrects
const [visible, setVisible] = useState<Todo[]>([]);
useEffect(() => {
  setVisible(todos.filter(t => t.status === filter));
}, [todos, filter]);

// FIX: inline if cheap
const visible = todos.filter(t => t.status === filter);

// FIX: memoize if genuinely expensive (measure first: console.time/timeEnd)
const visible = useMemo(
  () => expensiveFilter(todos, filter),
  [todos, filter]
);
```

**When is it expensive?** Measure it. `console.time('filter')` / `console.timeEnd('filter')`. If it's under 1ms, inline is fine. Most list filters on typical UI datasets (hundreds to low thousands of items) are sub-millisecond.

---

## 3. Resetting State on Prop Change

You want a "fresh start" when an identity prop changes. useEffect resets state one render too late (the component renders with stale state, then corrects).

```tsx
// BUG: renders with previous user's comment, then clears
function ProfilePage({ userId }: { userId: string }) {
  const [comment, setComment] = useState('');
  useEffect(() => { setComment(''); }, [userId]);
}

// FIX: React destroys and remounts, state is fresh from the start
function ProfilePage({ userId }: { userId: string }) {
  return <Profile key={userId} userId={userId} />;
}

function Profile({ userId }: { userId: string }) {
  const [comment, setComment] = useState(''); // always starts fresh
}
```

**Why key works**: React treats `<Profile key="1" />` and `<Profile key="2" />` as completely different component instances. It unmounts the old one and mounts a new one with fresh state.

**When to use key**: Whenever you'd write `useEffect(() => { resetAllState(); }, [identityProp])`. The key prop is the React-native way to express "this is a different thing now."

---

## 4. Event Logic in Effects

If something should happen because of a user action, it belongs in the event handler. Effects fire on mount and re-renders too, which causes phantom behavior.

```tsx
// BUG: shows notification on page load if product is already in cart
function ProductPage({ product, addToCart }) {
  useEffect(() => {
    if (product.isInCart) showNotification(`Added ${product.name}!`);
  }, [product]);

  function handleBuyClick() { addToCart(product); }
}

// FIX: notification happens exactly when the user clicks
function ProductPage({ product, addToCart }) {
  function handleBuyClick() {
    addToCart(product);
    showNotification(`Added ${product.name}!`);
  }
}
```

**The test**: "Should this happen if the user just navigated to this page without doing anything?" If no, it's event logic.

**Shared logic between handlers**: Extract a function, call from both handlers. Don't route through state + effect.

```tsx
function buyProduct() {
  addToCart(product);
  showNotification(`Added ${product.name}!`);
}

function handleBuyClick() { buyProduct(); }
function handleCheckoutClick() { buyProduct(); navigateTo('/checkout'); }
```

---

## 5. Effect Chains (Rube Goldberg Machines)

Multiple effects that trigger each other through state updates. Each one causes a re-render, and the chain is impossible to reason about or debug.

```tsx
// BUG: 4 effects = up to 4 extra renders per card placement
useEffect(() => {
  if (card?.gold) setGoldCount(c => c + 1);
}, [card]);

useEffect(() => {
  if (goldCount > 3) { setRound(r => r + 1); setGoldCount(0); }
}, [goldCount]);

useEffect(() => {
  if (round > 5) setGameOver(true);
}, [round]);

// FIX: derive what you can, compute all transitions in the event handler
const isGameOver = round > 5; // derived, not state

function handlePlaceCard(nextCard: Card) {
  if (isGameOver) throw Error('Game ended');
  setCard(nextCard);
  if (nextCard.gold) {
    if (goldCount < 3) {
      setGoldCount(goldCount + 1);
    } else {
      setGoldCount(0);
      setRound(round + 1);
    }
  }
}
```

**The signal**: If you see 2+ `useEffect` calls in the same component where one sets state that another watches, you have a chain. Collapse them.

---

## 6. Notifying Parent via Effect

An effect that calls `onChange` or `onUpdate` after local state changes. This means the parent re-renders after the child already rendered, causing unnecessary work.

```tsx
// BUG: child renders → effect fires → parent re-renders → child re-renders
function Toggle({ onChange }: { onChange: (v: boolean) => void }) {
  const [isOn, setIsOn] = useState(false);
  useEffect(() => { onChange(isOn); }, [isOn, onChange]);
  function handleClick() { setIsOn(!isOn); }
}

// FIX: notify in the same event, React batches both updates into one render
function Toggle({ onChange }: { onChange: (v: boolean) => void }) {
  const [isOn, setIsOn] = useState(false);
  function handleClick() {
    const next = !isOn;
    setIsOn(next);
    onChange(next);
  }
}

// BEST: fully controlled, no local state
function Toggle({ isOn, onChange }: { isOn: boolean; onChange: (v: boolean) => void }) {
  return <button onClick={() => onChange(!isOn)} />;
}
```

**Rule of thumb**: If two components need synchronized state, either lift state up or make the child fully controlled.

---

## 7. Passing Data Upward via Effect

Child fetches data, then pushes it up to parent through an `onFetched` callback in an effect. Data should flow down, not up.

```tsx
// BUG: data flows up (child fetches → effect → parent state → re-render)
function Parent() {
  const [data, setData] = useState(null);
  return <Child onFetched={setData} />;
}
function Child({ onFetched }) {
  const data = useSomeAPI();
  useEffect(() => { if (data) onFetched(data); }, [data, onFetched]);
}

// FIX: parent owns the fetch, passes data down
function Parent() {
  const data = useSomeAPI();
  return <Child data={data} />;
}
```

---

## 8. Fetching Without Cleanup (Race Conditions)

The classic: user types "hello" fast, responses for "h", "he", "hel", "hell", "hello" arrive out of order. Without cleanup, the UI shows whichever response arrived last, which might be "hel" instead of "hello".

```tsx
// BUG: race condition, stale results overwrite fresh ones
useEffect(() => {
  fetchResults(query).then(json => setResults(json));
}, [query]);

// FIX: cleanup flag ignores stale responses
useEffect(() => {
  let ignore = false;
  fetchResults(query).then(json => {
    if (!ignore) setResults(json);
  });
  return () => { ignore = true; };
}, [query]);

// BETTER: AbortController cancels the request entirely
useEffect(() => {
  const controller = new AbortController();
  fetchResults(query, { signal: controller.signal })
    .then(json => setResults(json))
    .catch(err => { if (err.name !== 'AbortError') throw err; });
  return () => controller.abort();
}, [query]);

// BEST: use a data fetching library
const { data } = useQuery({ queryKey: ['search', query], queryFn: () => fetchResults(query) });
```

---

## 9. Unstable Callback → Infinite Loop

The sneakiest useEffect bug. A parent passes a function prop that isn't memoized. Child puts it in a useEffect dep array. Every render creates a new function reference → effect fires → state update → re-render → new function → effect fires. Forever.

```tsx
// BUG: infinite loop
function Parent() {
  // New function reference every render
  const handleReady = (seekFn: (t: number) => void) => setSeek(() => seekFn);
  return <Player onReady={handleReady} />;
}

function Player({ onReady }: { onReady: (fn: (t: number) => void) => void }) {
  const seekTo = useCallback((time: number) => { /* ... */ }, []);
  useEffect(() => {
    onReady(seekTo); // fires every render because onReady is unstable
  }, [onReady, seekTo]);
}

// FIX: wrap in useCallback
function Parent() {
  const handleReady = useCallback(
    (seekFn: (t: number) => void) => setSeek(() => seekFn),
    []
  );
  return <Player onReady={handleReady} />;
}
```

**Detection**: Grep for `useEffect` where the dep array includes a prop that's a function. Every single one is a potential infinite loop unless the parent guarantees stable identity via `useCallback` or a `useState` setter (which is stable by default).

**Note**: `useState` setters (`setX`) are stable by default. They're safe in dep arrays without `useCallback`. But any function created inline in a component body is unstable.

---

## 10. App Initialization in Effect

Logic that should run once per app load, not once per component mount. In React 18+ Strict Mode, effects run twice in development, which can break auth tokens, double-send analytics, etc.

```tsx
// BUG: runs twice in dev, may invalidate auth token on second call
function App() {
  useEffect(() => {
    loadDataFromLocalStorage();
    checkAuthToken();
  }, []);
}

// FIX: module-level guard
let didInit = false;
function App() {
  useEffect(() => {
    if (!didInit) {
      didInit = true;
      loadDataFromLocalStorage();
      checkAuthToken();
    }
  }, []);
}

// ALSO FINE: module-level execution (runs before any component mounts)
if (typeof window !== 'undefined') {
  checkAuthToken();
  loadDataFromLocalStorage();
}
```

---

## 11. Seed-Once from Async Data via Effect

State that should be initialized from async data (a query, fetch, context) once, then owned by the user. This is NOT derived state because the user can change it independently after seeding. But it's also NOT a legitimate effect use because there's no external system to synchronize with.

The trap: it *looks* like derived state, so agents and developers make it fully derived, which kills interactivity. Then they overcorrect to `useEffect` with a ref guard, which works but adds unnecessary timing complexity (flash of empty state, effect runs after paint).

```tsx
// BUG: fully derived — combobox selection is always overwritten, user can never change it
function CategoryPicker({ categories, defaultType }) {
  const selectedIds = useMemo(() => {
    const match = categories?.find(c => c.type === defaultType);
    return match ? new Set([match.id]) : new Set();
  }, [categories, defaultType]);
  // selectedIds is recalculated every render, user's combobox changes are thrown away
}

// LESS WRONG but still unnecessary: useEffect seeds after paint (flash of empty state)
function CategoryPicker({ categories, defaultType }) {
  const [selectedIds, setSelectedIds] = useState<Set<string>>(new Set());
  const hasSeeded = useRef(false);
  useEffect(() => {
    if (!hasSeeded.current && categories && defaultType) {
      const match = categories.find(c => c.type === defaultType);
      if (match) { setSelectedIds(new Set([match.id])); hasSeeded.current = true; }
    }
  }, [categories, defaultType]);
}

// FIX: setState during render — synchronous, no flash, no effect
function CategoryPicker({ categories, defaultType }) {
  const [selectedIds, setSelectedIds] = useState<Set<string>>(new Set());
  const hasSeeded = useRef(false);

  if (!hasSeeded.current && categories && defaultType) {
    const match = categories.find(c => c.type === defaultType);
    if (match) {
      setSelectedIds(new Set([match.id]));
      hasSeeded.current = true;
    }
  }
  // After seeding, combobox onChange calls setSelectedIds normally
}
```

**How setState during render works**: React detects the `setState` call, bails out of the current render, and immediately re-renders with the new value. No effect timing, no flash of empty state.

**How to recognize**: State starts empty. Async data arrives. State should initialize from it once. After that, the user owns it (combobox, form field, toggle). The key question: "Can the user change this independently after the initial value?" If yes, it's seed-once.

---

## 12. Store ID, Not Object

When a list changes and you want to preserve selection, storing the full object forces you to use an effect to "adjust" it. Store the ID, derive the object.

```tsx
// BUG: effect needed to handle list changes, renders with stale selection
function List({ items }: { items: Item[] }) {
  const [selection, setSelection] = useState<Item | null>(null);
  useEffect(() => { setSelection(null); }, [items]);
}

// FIX: store ID, derive the object, no effect needed
function List({ items }: { items: Item[] }) {
  const [selectedId, setSelectedId] = useState<string | null>(null);
  const selection = items.find(item => item.id === selectedId) ?? null;
  // Bonus: if the item still exists in new list, selection is preserved
}
```
