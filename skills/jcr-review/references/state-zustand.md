# Zustand — State Management Patterns

## Why It Matters

Zustand is a minimal state library for React. Its small API makes it easy to adopt and equally easy to misuse — the most common problems are subscription over-rendering, monolithic stores, and hydration mismatches on Next.js. Most issues are silent performance cliffs rather than compile errors.

Apply this reference when reviewing React / Next.js / Tauri (web frontend) code. If the project uses a different state library (Redux Toolkit, Jotai, Recoil), the *principles* (narrow selection, co-locate concerns, predictable hydration) still apply; adapt them to the library's API.

## Principles

- **Always subscribe through a selector.** `useStore()` without a selector subscribes to the whole store.
- **Use `shallow` (or `useShallow`) for object / array selections.** A fresh object returned every render causes unnecessary rerenders.
- **Split stores by domain.** One store per concern, composed with slices. Avoid the "single global store for everything" pattern.
- **Derive values in selectors, not in stored state.** Stored derived values go stale and double the update surface.
- **For deep updates, reach for `immer` middleware.** Manual spread operators past two levels invite bugs.
- **Treat the store as ephemeral state.** Persist only what survives reload; persisting everything creates SSR and hydration pain.

## Checklist

### Selectors & Subscriptions

- [ ] `const state = useStore()` subscribing to the whole store — every mutation rerenders the component.
- [ ] Selector returning a fresh object/array without `shallow` or `useShallow`:
      `useStore(s => ({ a: s.a, b: s.b }))` — new reference each render.
- [ ] Selecting a computed value inside the selector on every call:
      `useStore(s => s.items.filter(...))` — recomputes each render. Move to a memoized selector or derive via `useMemo`.
- [ ] Accessing `useStore.getState()` inside a render — bypasses subscription; stale when store updates.
- [ ] `useStore.subscribe(fn)` called inside a component without a cleanup function — subscription leak across rerenders.

### Store Structure

- [ ] Single monolithic store mixing unrelated domains (auth + UI + feature data) — consumers rerender for unrelated changes.
- [ ] Slices not namespaced — selector like `s => s.user` ambiguous when multiple slices contribute.
- [ ] Actions co-mingled with state at the top level of the store object without any convention — hard to tell state from behavior.
- [ ] Derived value stored as state (`computed: items.length`) and manually kept in sync on every action — one missed action, stale data.

### Middleware

- [ ] `persist` middleware wrapping the entire store including functions, class instances, or promises — serialization silently drops them.
- [ ] `persist` without a `version` and `migrate` function — first schema change corrupts existing users' local state.
- [ ] `persist` with no `partialize` in a store containing sensitive data (tokens, PII) — leaks to `localStorage`.
- [ ] `devtools` middleware left enabled in production builds — exposes internal store shape.
- [ ] `immer` not used while doing deep nested updates (`set(s => ({ a: { ...s.a, b: { ...s.a.b, c: v } } }))`).
- [ ] `subscribeWithSelector` unused when the code relies on listening to a specific slice with equality checks.

### Async Actions

- [ ] Store action performing async work without loading / error state — UI has no way to show progress.
- [ ] Async action that calls `set` after the component unmounts — not always harmful (store is global) but worth noting if the action feeds component-only state.
- [ ] Async action that doesn't handle rejection — `console.error` at best, unhandled promise rejection at worst.
- [ ] Async action using stale store values from closure instead of `get()` — works until someone refactors and the bug surfaces.
- [ ] Fetching inside a component effect and setting store — duplicate fetches across mounts. Prefer store-owned action.

### SSR / Hydration (Next.js)

- [ ] `persist` middleware used in a server-rendered app without a hydration gate — `window`-dependent storage throws on SSR, or initial HTML differs from client.
- [ ] Store initialized with `Date.now()` / `Math.random()` / `crypto.randomUUID()` at module scope — server and client produce different values, triggering hydration mismatch.
- [ ] Store shared across requests on the server — in Next.js the module is reused, leaking one user's state into another's response. Use a per-request provider (`createStore` inside a client component) for request-scoped data.
- [ ] `useStore` called inside a Server Component — Zustand hooks are client-only.

### Testing

- [ ] Tests don't reset the store between cases — state leaks across tests, runs fail only in suite.
- [ ] Using the real store in unit tests when a mocked selector output would suffice — couples tests to implementation.
- [ ] No test for the rehydration path (`persist.onFinishHydration`, `_hasHydrated`) when the store uses `persist`.

### React Patterns Around Zustand

- [ ] Calling store actions inside `useEffect` with incorrect dependency array — either re-runs too often or not at all.
- [ ] Deriving local component state from store state with `useState(store.value)` — captured once, never updates. Use `useStore(s => s.value)`.
- [ ] Prop drilling values that already live in the store — pick them with a selector at the point of use.

## Good / Bad Examples

Bad — whole-store subscription:
```tsx
const state = useStore();          // any change rerenders
return <div>{state.user.name}</div>;
```

Good — narrow selector:
```tsx
const name = useStore((s) => s.user.name);
return <div>{name}</div>;
```

Bad — fresh object every render, no `shallow`:
```tsx
const { a, b } = useStore((s) => ({ a: s.a, b: s.b }));   // new ref → always rerender
```

Good — `useShallow`:
```tsx
import { useShallow } from 'zustand/react/shallow';

const { a, b } = useStore(useShallow((s) => ({ a: s.a, b: s.b })));
```

Bad — deep nested update without `immer`:
```ts
set((s) => ({
  form: { ...s.form, address: { ...s.form.address, city: newCity } },
}));
```

Good — `immer` middleware:
```ts
import { create } from 'zustand';
import { immer } from 'zustand/middleware/immer';

const useStore = create(immer((set) => ({
  form: { address: { city: '' } },
  setCity: (city) => set((s) => { s.form.address.city = city; }),
})));
```

Bad — monolithic store:
```ts
const useStore = create(() => ({
  // auth
  user: null, login: () => { ... }, logout: () => { ... },
  // ui
  modal: null, openModal: () => { ... },
  // cart
  items: [], addItem: () => { ... },
  // ...
}));
```

Good — slices pattern:
```ts
const useStore = create<AuthSlice & UiSlice & CartSlice>()((...a) => ({
  ...createAuthSlice(...a),
  ...createUiSlice(...a),
  ...createCartSlice(...a),
}));
```

Bad — `persist` without partialize, leaking tokens:
```ts
const useAuth = create(
  persist((set) => ({ user: null, accessToken: '', login: () => { ... } }),
          { name: 'auth' })
);
// accessToken is now in localStorage.
```

Good — `partialize` whitelist:
```ts
persist(
  (set) => ({ user: null, accessToken: '', login: () => { ... } }),
  {
    name: 'auth',
    partialize: (s) => ({ user: s.user }),   // token kept in memory only
  }
);
```

Bad — Next.js module-scoped store with per-request data:
```ts
// store.ts — imported by server components, shared across requests
export const useStore = create(() => ({ cart: [] }));
```

Good — per-request provider:
```tsx
// CartStoreProvider.tsx (client component)
'use client';
import { createStore } from 'zustand';
// Create a new store instance inside a React context provider,
// scoped to one request / one user session.
```

## Anti-patterns

- **Mega Store** — One store containing every domain, every feature flag, every UI ephemeral. Split by concern.
- **Hidden Global State** — Using `useStore.getState()` from outside React (services, utilities) to read or write state. Breaks reactivity and obscures data flow.
- **Derive-and-Store** — Computing a value in an action and storing it. Now you have two sources of truth; the one in state is always one action behind.
- **Persist Everything** — `persist(store)` with no partialize. Ephemeral UI state ends up in `localStorage`; tokens and PII leak; schema changes break users.
- **Selector Explosion** — One component doing 8 individual `useStore(s => s.x)` calls. Consolidate with a `useShallow` object selector.
- **Stale Closure Action** — Async action captures store values from the outer closure instead of calling `get()` at await boundaries.
- **Effect-Driven Fetching** — Every screen mounts an effect that calls `store.fetchX()`, resulting in duplicate requests. Own the fetch inside the store or with a data-fetching library (SWR / React Query).
- **Test Pollution** — Tests mutate the store and don't reset it, causing order-dependent failures.
