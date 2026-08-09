---
name: typescript-react
description: Use when writing or updating React or TypeScript component files. Patterns for hooks, types, composition, accessibility, and testing.
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
- `useEffect` should have a clear cleanup story. If the effect's job is "do X once on mount," prefer derived state or an event handler when one fits.
- `useCallback` / `useMemo` only when there's a real reference-stability or expensive-compute reason. Don't memoize prophylactically.
- Custom hooks: same naming rule (`useThing`); same rules of hooks apply.

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

## Accessibility

- Semantic HTML first: `<button>`, `<a>`, `<nav>`, `<main>`. Don't recreate them with `<div onClick>`.
- Every interactive element is keyboard-reachable. Focus styles never hidden globally.
- `aria-*` only when semantic HTML can't express the role.
- Images have `alt`. Decorative images use `alt=""`.

## Styling

Match the project's existing approach and don't introduce a new system. Where the project pairs Tailwind with CSS custom properties (as ContextMatrix `web/` does), that pair is the whole vocabulary: utility classes plus `var(--token)` / `text-[var(--token)]`. No CSS modules, no styled-components, and never a hardcoded hex literal in a component - palettes are swappable at runtime, so hex in a `.tsx` breaks every non-default palette. Reach for `@apply` only when the project already does.

## Testing

- Vitest + React Testing Library (`@testing-library/react`, `@testing-library/jest-dom`, jsdom). Run with `npx vitest run` from the frontend root.
- Query by accessible role/label (`getByRole('button', { name: /submit/i })`), not by class name.
- Avoid `data-testid` unless nothing accessible works.
- Behavioral tests, not snapshots. Snapshots become noise on a fast-evolving UI.
- Mock at the module boundary with `vi.mock`: the project's typed API wrapper (`vi.mock('../../api/client', ...)`) for server calls, the owning hook for shared state. Do not add MSW or another network-interception dependency - this project has none.

## Quick red flags

| Red flag                                          | Why it's wrong                                              |
| ------------------------------------------------- | ----------------------------------------------------------- |
| `any` in a function signature or prop type        | Loses type safety                                           |
| `as Foo` to silence a type error                  | Bypasses the checker; fix the underlying type               |
| `<div onClick>` for an interactive element        | Not keyboard-accessible; use `<button>`                     |
| `useEffect` with `[]` doing setup work            | Often a sign that derived state or an event handler fits    |
| Class component in a new file                     | Use function components                                     |
| Index used as `key` in a `.map`                   | Breaks reconciliation when list reorders                    |
| `useCallback` / `useMemo` everywhere              | Premature; profile before adding                            |
| Missing cleanup in subscription/timer effect      | Leak                                                        |
