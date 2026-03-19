---
name: make-it-readable
description: Code readability cleanup pass. Makes code skimmable by reducing state, narrowing types, removing unnecessary abstractions, preferring asserts over defensive code, using discriminated unions, early returns, and fewer lines. Run on changed code before every PR. Use when reviewing diffs, refactoring, cleaning up code, or when code feels over-engineered, too defensive, or hard to follow.
---

# Make It Readable

Code should be skimmable. A reader should understand what a function does without tracing through abstractions, mentally unwinding defensive checks, or wondering which of 8 optional parameters actually matter.

This skill is a cleanup pass. Run it on changed code before every PR.

## The Rules

### 1. Write skimmable code

A function's purpose should be obvious from a glance. If you need to read every line to understand what it does, it's too complex.

```ts
// BEFORE: have to trace through the whole thing
function processUserData(data: Record<string, unknown>) {
  const result: Record<string, unknown> = {};
  const keys = Object.keys(data);
  for (let i = 0; i < keys.length; i++) {
    const key = keys[i];
    const value = data[key];
    if (typeof value === 'string') {
      result[key] = value.trim();
    } else if (typeof value === 'number') {
      result[key] = Math.round(value);
    } else {
      result[key] = value;
    }
  }
  return result;
}

// AFTER: intent is obvious
function cleanUser(user: { name: string; age: number }) {
  return { name: user.name.trim(), age: Math.round(user.age) };
}
```

The best way to make code skimmable is to make it short, typed, and obvious. Most of the rules below are specific ways to achieve this.

### 2. Minimize possible states

Every argument is a dimension of state. Every optional field doubles the states your function can be in. Reduce arguments, remove optional fields, narrow types.

```ts
// BEFORE: 2^3 = 8 possible states from 3 booleans
function createNotification(
  message: string,
  isError: boolean,
  isUrgent: boolean,
  isDismissible: boolean
) { /* ... */ }

// AFTER: one argument, one state space
type Notification =
  | { type: "info"; message: string }
  | { type: "error"; message: string; urgent: boolean }
  | { type: "toast"; message: string; autoDismissMs: number };

function createNotification(notification: Notification) { /* ... */ }
```

### 3. Use discriminated unions

A discriminated union tells you exactly which fields exist for each case. No guessing, no runtime checks for fields that might be there.

```ts
// BEFORE: which fields are set? who knows
type ApiResponse = {
  data?: User[];
  error?: string;
  loading: boolean;
  retryAfter?: number;
};

// is this an error with data? loading with an error? anything goes
function render(response: ApiResponse) {
  if (response.loading) return <Spinner />;
  if (response.error) return <Error msg={response.error} />;
  if (response.data) return <UserList users={response.data} />;
  return null; // what state is this?
}

// AFTER: exactly 3 states, nothing else is possible
type ApiResponse =
  | { status: "loading" }
  | { status: "error"; error: string; retryAfter?: number }
  | { status: "success"; data: User[] };

function render(response: ApiResponse) {
  switch (response.status) {
    case "loading": return <Spinner />;
    case "error": return <Error msg={response.error} />;
    case "success": return <UserList users={response.data} />;
  }
}
```

### 4. Exhaustively handle all variants, fail on unknown

When you switch on a discriminated union, handle every case. Use `never` to catch unhandled variants at compile time. If you encounter a type you didn't expect at runtime, fail loudly.

```ts
function assertNever(x: never): never {
  throw new Error(`Unexpected variant: ${JSON.stringify(x)}`);
}

function getStatusColor(status: TaskStatus): string {
  switch (status) {
    case "pending": return "yellow";
    case "running": return "blue";
    case "done": return "green";
    case "failed": return "red";
    default: assertNever(status); // compile error if a case is missing
  }
}
```

### 5. Don't write defensive code

Trust your types. If the type says it's a `string`, don't check if it's `null`. Defensive code hides bugs by silently handling states that should never exist. If the type is wrong, fix the type.

```ts
// BEFORE: defensive, hides bugs
function getDisplayName(user: User): string {
  if (!user) return "Unknown";
  if (!user.name) return "Unknown";
  if (typeof user.name !== "string") return "Unknown";
  return user.name;
}

// AFTER: trust the type, if user is null that's a bug upstream
function getDisplayName(user: User): string {
  return user.name;
}
```

If your types are wrong and `user` actually can be null, the fix is to make the type `User | null` and handle it at the boundary where the data enters your system, not inside every function that touches it.

### 6. Assert at boundaries, be strict about parameters

When data enters your system, don't silently fill in defaults for missing fields. That hides bugs. If the config is missing `apiUrl`, that's a deployment problem — crash, don't invent a default URL that will fail differently later.

```ts
// BEFORE: silently papers over missing config with defaults
function getApiUrl(config: Partial<Config>): string {
  return config.apiUrl ?? "https://default.api";
}

// AFTER: if it's missing, that's a bug — find it now
function getApiUrl(config: Config): string {
  return config.apiUrl; // Config requires apiUrl, validated at load time
}
```

Same principle for function parameters. Be opinionated: if your function needs a value, require it. Don't make things optional and then scramble to handle the missing case.

```ts
// BEFORE: optional param, defensive fallback
function getConversation(id?: string) {
  const conversationId = id ?? store.getState().currentConversationId;
  assert(conversationId, "No conversation ID");
  return db.get(conversationId);
}

// AFTER: caller is explicit, function is simple
function getConversation(id: string) {
  return db.get(id);
}
```

If you're loading untyped data (API response, JSON file, URL params), validate the shape once at the boundary with asserts or a schema library, then trust the types everywhere downstream.

### 7. Remove anything not strictly required

If a line doesn't contribute to the current behavior, delete it. Dead code, unused imports, commented-out blocks, console.logs, TODO comments for things that will never happen. Don't commit it.

```ts
// BEFORE
import { formatDate, formatCurrency, formatPercentage } from "./format";

function renderPrice(price: number) {
  // TODO: add currency conversion later
  // const converted = convertCurrency(price, userCurrency);
  const formatted = formatCurrency(price);
  // console.log("price:", formatted);
  return formatted;
}

// AFTER
import { formatCurrency } from "./format";

function renderPrice(price: number) {
  return formatCurrency(price);
}
```

### 8. Bias for fewer lines

Fewer lines means less to read, less to break, less to review. This doesn't mean minify your code. It means don't spread a simple operation across 10 lines when 3 are clear.

```ts
// BEFORE: 12 lines
function getUserIds(users: User[]): string[] {
  const ids: string[] = [];
  for (const user of users) {
    if (user.active) {
      ids.push(user.id);
    }
  }
  return ids;
}

// AFTER: 1 line, equally clear
const getActiveUserIds = (users: User[]) =>
  users.filter(u => u.active).map(u => u.id);
```

### 9. No complex or clever code

If it needs a comment to explain, it's too clever. Straightforward code that reads like prose beats elegant code that reads like a puzzle.

```ts
// BEFORE: clever bitwise check for even/odd
const isEven = (n: number) => !(n & 1);

// AFTER: obvious
const isEven = (n: number) => n % 2 === 0;
```

```ts
// BEFORE: clever reduce
const grouped = items.reduce<Record<string, Item[]>>((acc, item) => {
  (acc[item.category] ??= []).push(item);
  return acc;
}, {});

// AFTER: Object.groupBy or a plain loop
const grouped = Object.groupBy(items, item => item.category);
```

### 10. Don't over-extract functions

Abstraction has a cost: the reader has to jump to another location to understand what happens. If a function is called once and is only a few lines, inline it.

```ts
// BEFORE: 4 tiny functions called once each
function validateEmail(email: string) { return email.includes("@"); }
function validatePassword(pw: string) { return pw.length >= 8; }
function validateName(name: string) { return name.trim().length > 0; }
function validateForm(form: SignupForm) {
  return validateEmail(form.email) &&
    validatePassword(form.password) &&
    validateName(form.name);
}

// AFTER: one place, readable top to bottom
function validateForm(form: SignupForm) {
  return (
    form.email.includes("@") &&
    form.password.length >= 8 &&
    form.name.trim().length > 0
  );
}
```

Extract when: the function is reused, or the block is long enough that it obscures the surrounding logic. Don't extract when: it just moves 3 lines somewhere else.

### 11. Early returns

Guard clauses at the top, happy path at the bottom. No deep nesting.

```ts
// BEFORE: nested pyramid
function processOrder(order: Order) {
  if (order) {
    if (order.items.length > 0) {
      if (order.status === "pending") {
        // ... actual logic buried 3 levels deep
        return submitOrder(order);
      } else {
        throw new Error("Order not pending");
      }
    } else {
      throw new Error("Empty order");
    }
  } else {
    throw new Error("No order");
  }
}

// AFTER: flat, guards at top, logic at bottom
function processOrder(order: Order) {
  assert(order, "No order");
  assert(order.items.length > 0, "Empty order");
  assert(order.status === "pending", "Order not pending");

  return submitOrder(order);
}
```

### 12. Assert, don't try-catch or default

When you expect something to exist, assert it. A try-catch or default value hides the fact that something went wrong. Fail fast, fix the bug.

```ts
// BEFORE: silently returns undefined behavior
function getUser(id: string): User {
  try {
    return db.users.find(u => u.id === id)!;
  } catch {
    return { id, name: "Unknown", email: "" } as User;
  }
}

// BEFORE: silent null propagation
function getUserEmail(id: string): string {
  return db.users.find(u => u.id === id)?.email ?? "";
}

// AFTER: crash means a bug, you'll find it immediately
function getUser(id: string): User {
  const user = db.users.find(u => u.id === id);
  assert(user, `User not found: ${id}`);
  return user;
}
```

Use try-catch for things that are *expected* to fail (network requests, file I/O, user input). Use asserts for things that should *never* fail (looking up data you know exists, accessing config that was validated at startup).

### 13. Keep argument count low, no overrides

Functions with many arguments are hard to call, hard to read, and hard to change. If you're passing overrides "just in case," you're designing for a future that probably won't come.

```ts
// BEFORE: 6 args, 3 are overrides "for flexibility"
function sendEmail(
  to: string,
  subject: string,
  body: string,
  from?: string,
  replyTo?: string,
  headers?: Record<string, string>
) { /* ... */ }

// AFTER: required args only, expand when you actually need to
function sendEmail(to: string, subject: string, body: string) { /* ... */ }
```

If you genuinely need many parameters, use a single options object, but still don't make things optional unless they're truly optional.

### 14. Don't make required arguments optional

If your function breaks without an argument, that argument is required. Making it optional with a default value hides the dependency and creates an implicit contract.

```ts
// BEFORE: looks optional, but everything breaks without userId
function fetchProfile(userId?: string) {
  const id = userId ?? getCurrentUserId();
  return api.get(`/profiles/${id}`);
}

// AFTER: caller must be explicit
function fetchProfile(userId: string) {
  return api.get(`/profiles/${userId}`);
}
```

## Quick Reference

| Rule | Smell | Fix |
|------|-------|-----|
| Skimmable | Can't understand function at a glance | Simplify, rename, shorten |
| Minimize states | Many boolean params, wide union of optional fields | Discriminated union, fewer args |
| Discriminated unions | Optional fields that depend on each other | Tagged union with `status`/`type` field |
| Exhaustive handling | `default:` that returns a fallback silently | `assertNever` in default, handle every case |
| No defensive code | `if (!x) return fallback` for typed values | Trust the type, fix upstream if wrong |
| Assert at boundary | Defaults/fallbacks when loading data | `assert()` or schema validation |
| Remove dead code | Commented code, unused imports, TODOs | Delete it |
| Fewer lines | Simple logic spread across many lines | Condense, use standard library |
| No clever code | Needs a comment to explain | Rewrite to be obvious |
| Don't over-extract | One-use 3-line helper function | Inline it |
| Early returns | Nested if/else pyramid | Guard clauses at top |
| Assert don't catch | try-catch around code that "should work" | `assert()` |
| Low arg count | 5+ params, overrides "for flexibility" | Remove overrides, use options object if needed |
| No fake optionals | `param?: T` with a required default | Make it required |

## Review Checklist

When reviewing a diff:

- [ ] Can I understand each function at a glance?
- [ ] Are there boolean flags that should be a discriminated union?
- [ ] Is every switch/if-else chain exhaustive?
- [ ] Is there defensive code for types that guarantee the value exists?
- [ ] Is data validated at the boundary with asserts or schemas?
- [ ] Is every line in the diff necessary for the current change?
- [ ] Can any multi-line block be expressed more concisely?
- [ ] Is there clever code that needs a comment to explain?
- [ ] Are there extracted functions that are called once and are short?
- [ ] Are guard clauses at the top with the happy path at the bottom?
- [ ] Are there try-catches or defaults hiding expected-to-exist values?
- [ ] Are there optional parameters that are actually required?
- [ ] Is the argument count low with no speculative overrides?
