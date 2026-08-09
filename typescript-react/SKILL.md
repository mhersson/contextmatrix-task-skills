---
name: typescript-react
description: Use when writing or updating React or TypeScript component files, including adding UI to an app that already has a design system, styling with its existing tokens, or wiring a component to streamed or remote data.
---

You are a senior React/TypeScript engineer. Match the surrounding code's patterns first; introduce new ones only when the existing approach is clearly worse for the task.

## Components

**Iron law:** function components with hooks. No class components.

- Co-locate state with the component that owns it. Lift only when truly shared.
- Avoid prop drilling beyond two levels — use context, composition, or component slots.
- Component prop types live next to the component as a named `interface` or `type`.
- Composition over configuration: `<Card><Card.Header>` over `<Card header={...}>` for structural variation.

## Hooks

- Respect the rules: hooks only at top level, only from React functions.
- `useEffect` synchronizes with a system outside React: subscriptions, timers, imperative DOM and focus work, non-React browser APIs. Every one returns a cleanup.
- `useCallback` / `useMemo` only when there's a real reference-stability or expensive-compute reason. Don't memoize prophylactically.
- Custom hooks: same naming rule (`useThing`); same rules of hooks apply.


### Not an effect

Before writing `useEffect`, check it is not one of these:

| You wrote                                                   | Do this instead                                                 |
| ----------------------------------------------------------- | --------------------------------------------------------------- |
| effect that `setState`s a value computable from props/state | compute it during render: `const fullName = first + ' ' + last` |
| effect gated on a `submitted` / `clicked` flag              | do the work in the event handler that set the flag              |
| effect that resets state when a prop changes                | pass a `key` and let React remount the subtree                  |

- Depend on primitives, not objects: `[card.id]`, not `[card]`. A server payload is a new object identity on every fetch or event.
- Derive the boolean outside the effect (`const isMobile = width < 768`) so it re-runs on the transition, not on every intermediate value.

### Stale closures in long-lived subscriptions

A callback registered once and kept alive (SSE `subscribe`, `addEventListener`, `setInterval`, WebSocket `onmessage`) captures the state it saw at registration. It goes stale the moment that state changes.

- Update from previous state with the functional form, never the captured variable:

  ```ts
  // BAD - `events` is frozen at subscribe time
  useEffect(() => subscribe((e) => setEvents([...events, e])), []);

  // GOOD - always operates on the latest value
  useEffect(() => subscribe((e) => setEvents((curr) => [...curr, e])), []);
  ```

- Do not "fix" it by adding the state to the dependency array. That tears down and re-opens the subscription on every update.
- Need the latest props or callbacks inside the subscription? On React 19.2+, wrap them in `useEffectEvent` and keep them out of the deps.
- The effect returns the unsubscribe. A subscription effect with no cleanup is a leak.

## TypeScript

**Iron law:** prefer narrow types over `any`. Use `unknown` for genuinely unknown data; narrow at the boundary.

- Discriminated unions for state machines:

  ```ts
  // GOOD
  type Status =
    | { kind: 'idle' }
    | { kind: 'loading' }
    | { kind: 'error'; message: string }
    | { kind: 'ok'; data: User };
  ```

- Avoid `as` type assertions. They bypass the checker; runtime guards or generic constraints are almost always better.
- Don't widen with `as any` to silence an error. Fix the type or the call site.
- Generics for genuine polymorphism, not "I don't want to type this."

## State management

- Component state for component-local concerns.
- Lifted state for sibling-shared concerns.
- Context only when prop drilling is genuinely painful (3+ levels) AND the value changes rarely. Frequently-changing context causes wide re-renders.
- A library (Zustand, Redux Toolkit, etc.) only when the project already uses one — match it.

## Async and streaming UI

**Iron law:** a view backed by remote or streamed data renders four states - loading, error, empty, content. Shipping only the content branch is an incomplete feature, not a shortcut.

```tsx
if (isLoading) return <CardListSkeleton />;
if (error) return <ErrorState message="Failed to load cards" onRetry={refetch} />;
if (cards.length === 0) return <EmptyState message="No cards yet" action={onCreate} />;
return <CardList cards={cards} />;
```

- Skeletons for content-shaped loading, spinners only for indeterminate actions. Mark the region: `<div role="status" aria-busy="true">Loading cards</div>` - visually hidden text is fine.
- Empty states name what is missing and offer the action that fixes it. Never a blank panel.
- Errors are recoverable: a message plus a retry affordance.
- A response that arrives after a newer request must not win. Key the result to the request (an incrementing id, or an `AbortController` you abort in the effect cleanup) and drop anything stale.
- Updates that arrive with no user action (SSE, polling, background refresh) are silent to a screen reader unless announced:

  | Region                                   | Announced         | Use for                    |
  | ---------------------------------------- | ----------------- | -------------------------- |
  | `role="status"` / `aria-live="polite"`   | at the next pause | saved, synced, N new items |
  | `role="alert"` / `aria-live="assertive"` | immediately       | errors, connection lost    |

  Keep the live region small and separate from the feed. A live region wrapped around a list that React re-renders wholesale can re-announce every item.

- Prepending to a scrolled list moves the content under the reader. Anchor on the pre-prepend scroll height, or pin to the bottom only when the user was already at the bottom.
- Overlays (modal, dropdown, popover) move focus in on open, keep focus inside while open, restore it to the trigger on close, and close on Escape.

## Accessibility

- Semantic HTML first: `<button>`, `<a>`, `<nav>`, `<main>`. Don't recreate them with `<div onClick>`.
- Every interactive element is keyboard-reachable. Focus styles never hidden globally.
- `aria-*` only when semantic HTML can't express the role.
- Images have `alt`. Decorative images use `alt=""`.

## Styling

Match the project's existing approach and don't introduce a new system. Where the project pairs Tailwind with CSS custom properties (as ContextMatrix `web/` does), that pair is the whole vocabulary: utility classes plus `var(--token)` / `text-[var(--token)]`. No CSS modules, no styled-components. Reach for `@apply` only when the project already does.

**Iron law:** components reference tokens, never literal values.

- Find the token source first (`index.css`, `tailwind.config.*`, a theme file) and read the semantic names before writing a single class. Never infer a color from a screenshot or from a sibling's rendered output.
- No hardcoded `#rrggbb`, `rgb()`, or `hsl()` in a component, including Tailwind arbitrary values like `text-[#7fbbb3]`. A literal color passes typecheck and lint, then breaks silently under theme or palette switching - the exact failure the token layer exists to prevent.
- Spacing comes off the scale. `padding: 13px` and `margin-top: 2.3rem` are off-scale.
- If a color, radius, or spacing value you need has no token, that is a design-system question. Say so on the card. Do not invent one.
- Never convey state by color alone. Pair it with an icon, a label, or a shape.
- Every component works in both light and dark. The theme is the user's runtime toggle, not a design decision.

## Testing

- Vitest + React Testing Library (`@testing-library/react`, `@testing-library/jest-dom`, jsdom). Run with `npx vitest run` from the frontend root.
- Query by accessible role/label (`getByRole('button', { name: /submit/i })`), not by class name.
- Avoid `data-testid` unless nothing accessible works.
- Behavioral tests, not snapshots. Snapshots become noise on a fast-evolving UI.
- Mock at the module boundary with `vi.mock`: the project's typed API wrapper (`vi.mock('../../api/client', ...)`) for server calls, the owning hook for shared state. Do not add MSW or another network-interception dependency - this project has none.

## Quick red flags

Ordered worst-first. Severity tiers match the `code-review` skill; anything above Nit is a defect, not a style preference.

| Severity  | Red flag                                                    | Why it's wrong                                                                                                                                                                                                                                      |
| --------- | ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Important | Component defined inside another component                  | New component type every render, so React remounts it and destroys its state. Symptoms: input loses focus on every keystroke, animations restart, effect cleanup/setup runs on every parent render, scroll position resets. Hoist it and pass props. |
| Important | Handler registered in a `[]` effect reading state directly  | Stale closure - it never sees an update. Use the functional `setState` form.                                                                                                                                                                        |
| Important | Missing cleanup in a subscription or timer effect           | Leak                                                                                                                                                                                                                                                |
| Important | `any` in a function signature or prop type                  | Loses type safety                                                                                                                                                                                                                                   |
| Minor     | `as Foo` to silence a type error                            | Bypasses the checker; fix the underlying type                                                                                                                                                                                                       |
| Minor     | `<div onClick>` for an interactive element                  | Not keyboard-accessible; use `<button>`                                                                                                                                                                                                             |
| Minor     | `useEffect` that only mirrors props into state              | Derive during render instead                                                                                                                                                                                                                        |
| Minor     | Index used as `key` in a `.map`                             | Breaks reconciliation when the list reorders                                                                                                                                                                                                        |
| Minor     | Hardcoded hex, `rgb()`, or an off-scale spacing value       | Bypasses the design tokens; breaks theme and palette switching                                                                                                                                                                                      |
| Minor     | `{count && <Badge />}` where `count` is a number            | Renders a literal `0`. Use `count > 0 ? <Badge /> : null`                                                                                                                                                                                            |
| Minor     | `useState(expensiveInit())`                                 | The initializer runs on every render. Use `useState(() => expensiveInit())` for `localStorage` reads, `JSON.parse`, index building                                                                                                                   |
| Minor     | Class component in a new file                               | Use function components                                                                                                                                                                                                                             |
| Nit       | `forwardRef` or `<Context.Provider>` in new code on React 19 | `ref` is a regular prop; render `<Context value={...}>`. Don't sweep existing call sites                                                                                                                                                             |
| Nit       | `useCallback` / `useMemo` everywhere                        | Premature; profile before adding                                                                                                                                                                                                                    |
