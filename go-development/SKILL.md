---
name: go-development
description: Use when implementing or modifying Go source files, or when choosing between concurrency primitives, error-wrapping strategies, or a legacy versus current stdlib idiom.
---

You are a senior Go engineer. Match the surrounding code's style first; introduce new patterns only when the existing one is clearly worse for the task at hand.

## Modern Go

**Read `go.mod`'s `go` directive before using anything below.** Never reach for an API newer than it, and never bump the directive to unlock one - that is a separate decision the card must authorize.

| Instead of                                   | Write                                          | Since |
| -------------------------------------------- | ---------------------------------------------- | ----- |
| `for i := 0; i < n; i++`                     | `for i := range n`                             | 1.22  |
| hand-rolled `minInt` / `maxInt`              | `min(a, b)` / `max(a, b, c)`                   | 1.21  |
| `for k := range m { delete(m, k) }`          | `clear(m)`                                     | 1.21  |
| `sort.Slice` / `sort.Strings`                | `slices.SortFunc` / `slices.Sort`              | 1.21  |
| a manual find loop                           | `slices.Contains` / `slices.IndexFunc`         | 1.21  |
| `append([]T(nil), s...)`, a map-copy loop    | `slices.Clone` / `maps.Clone`                  | 1.21  |
| `if s == "" { s = def }`                     | `cmp.Or(s, def)`                               | 1.22  |
| `sync.Once` + a package-level var            | `sync.OnceValue(func() T { ... })`             | 1.21  |
| `wg.Add(1)` + `go func(){ defer wg.Done() }` | `wg.Go(func(){ ... })`                         | 1.25  |
| `HasPrefix` then `TrimPrefix`                | `v, ok := strings.CutPrefix(s, p)`             | 1.20  |
| a multierror library, manual joining         | `errors.Join(err1, err2)`                      | 1.20  |
| `var pe *PathError; errors.As(err, &pe)`     | `pe, ok := errors.AsType[*PathError](err)`     | 1.26  |
| `math/rand` + `rand.Seed`                    | `math/rand/v2` (`IntN`, auto-seeded)           | 1.22  |
| `filepath.Join(base, userInput)`             | `os.OpenRoot(base)` then `root.Open(input)`    | 1.24  |
| `json:"x,omitempty"` on `time.Time` / `bool` | `json:"x,omitzero"`                            | 1.24  |
| `context.Background()` inside a test         | `t.Context()`                                  | 1.24  |
| `for i := 0; i < b.N; i++`                   | `for b.Loop()`                                 | 1.24  |
| `time.Sleep` to order a concurrency test     | `synctest.Test(t, ...)` + `synctest.Wait()`    | 1.25  |
| `v := v` loop-variable shadow copy           | delete it - each iteration already has its own | 1.22  |

Run `go fix ./...` before hand-editing: Go 1.26 rewrote it around the modernize analyzers and it applies a safe subset for you.

Do not modernize code the card did not ask you to touch. Fix the idiom on lines you are already editing; leave the rest.

## Formatting

- Format with `gofumpt -w .`, not `gofmt`. `gofumpt -l .` must come back empty before you call the work done - this holds even in a repo whose `make fmt` target still runs `go fmt`.
- Run `go fix ./...` before committing; it rewrites legacy stdlib forms into the current idiom.
- `golangci-lint` runs with `fix: false` and never rewrites the tree. Fix findings by hand.

## Errors

**Iron law:** every error is handled with intent, and handled exactly once - **logged or returned, never both.** No bare propagation, no swallowing. Logging on the way up emits one line per layer for a single failure, burying the real site in aggregated logs. Log only where the error stops.

Decide the strategy before you write the return:

1. **Crossing a system boundary** (HTTP response body, RPC reply, stored payload)? Format with `%v` so internal paths and types do not leak outward; log the full `%w` chain on your side of the boundary.
2. **Caller needs to branch on this condition?** Sentinel error matched with `errors.Is`, or a typed error matched with `errors.As` (`errors.AsType[T]` on Go 1.26+), wrapped with `%w`.
3. **Caller only needs debugging context?** `fmt.Errorf("load config: %w", err)`.

There is no fourth branch. Even when you have nothing to add, wrap with your own operation name rather than bare `return err` - the caller's stack is not in the message.

- Wrap with context: `fmt.Errorf("operation: %w", err)`. Never `return err` without context.
- Sentinel errors at package top for expected conditions: `var ErrNotFound = errors.New("not found")`. Compare with `errors.Is`.
- Custom error types for rich context: implement `Error()` (and `Unwrap()` if wrapping). Match with `errors.As`.
- Lowercase, no punctuation in messages: `"connect to database"`, not `"Failed to connect to database."`
- Never silently drop an error on a path that can fail meaningfully. Deliberate discards are written `_ = f.Close()` - that explicit form is required by `errcheck` and is the house idiom for deferred cleanup (`Close`, `Remove`, best-effort notifications). Anywhere else, handle or propagate.
- Integration seams (HTTP boundaries, external service calls) are where errors stop, so that is where you log them - and having logged, do not also return them.
- Validate constructor args; return errors for invalid configuration rather than silently defaulting. HTTP query params return 400 on parse failure.

  ```go
  // BAD
  json.Unmarshal(data, &v)

  // GOOD
  if err := json.Unmarshal(data, &v); err != nil {
      return fmt.Errorf("unmarshal response: %w", err)
  }
  ```

## Interfaces

**Iron law:** accept interfaces, return concrete types. Define interfaces at the point of use (consumer-side), not where they're implemented.

- Keep them small (1-2 methods). 5+ methods is a smell — split.
- Return concrete `*Client`, not `ClientInterface`. Constructor consumers can wrap if needed.
- No pointers to interfaces: `Handle(r io.Reader)`, never `Handle(r *io.Reader)`.
- Avoid `any` / `interface{}` in signatures. Use generics or concrete types.

## Context

- `context.Context` is the first parameter of any function that does I/O or that may need to be canceled. Pass through; don't store in structs.
- Honor cancellation: select on `ctx.Done()` in long loops; check after blocking calls.
- `context.Background()` and `context.TODO()` belong in `main` and tests; library code receives context from callers.

## Concurrency

**Iron law:** before you launch a goroutine, know when it stops.

### Pick the primitive

| Situation                                       | Use                                | Note                                                  |
| ----------------------------------------------- | ---------------------------------- | ----------------------------------------------------- |
| Handing a value from one goroutine to another   | channel                            | transfers ownership; declare direction in signatures  |
| Protecting struct fields                        | `sync.Mutex` / `sync.RWMutex`      | never hold across I/O; never upgrade `RLock` to `Lock` |
| A counter or a flag                             | `atomic.Int64` / `atomic.Bool`     | typed atomics, not raw `int32` + `atomic.StoreInt32`  |
| Fan-out where the first failure aborts the rest | `errgroup.WithContext(ctx)`        | cancels siblings; `g.Wait()` returns the first error  |
| Fan-out with a concurrency cap                  | `errgroup.Group` + `SetLimit(n)`   | replaces a hand-rolled worker pool                    |
| Fan-out where errors genuinely do not matter    | `sync.WaitGroup` + `wg.Go` (1.25+) | no error propagation, no cancellation                 |
| Deduplicating identical in-flight work          | `singleflight.Group`               | prevents cache stampedes                              |
| One-time initialization                         | `sync.OnceValue` (1.21+)           | not `sync.Once` plus a package-level var              |

`errgroup` and `singleflight` live in `golang.org/x/sync`. Use them when the module already requires it; do not add the dependency for a single call site.

**Only the sender closes a channel** - closing from the receive side panics any sender still writing. Declare direction (`chan<- T`, `<-chan T`) in signatures so the compiler enforces ownership.

On Go before 1.25, call `wg.Add` before `go`, never inside the goroutine - `Wait` can otherwise return early.

- Channels orchestrate; mutexes serialize. Document what each mutex protects.
- Every goroutine has an explicit shutdown path or a parent context that cancels it.
- Completion signals (done channels, WaitGroup.Done) fire after all side effects, not before.
- When replacing a slot (channel field, cancel func), close or cancel the old one first.
- Never start goroutines in library code without caller control — expose a `Start(ctx)` method instead of `go` in a constructor.
- Buffered channels need justification (a known producer/consumer ratio); unbuffered is the default.
- `sync.WaitGroup` for fan-out; pass by pointer, never by value.
- Don't copy a `sync.Mutex` after first use (no value-receiver methods on types that embed one).
- `time.After` in loops leaks timers — use `time.NewTimer` with `Stop()` and drain.

## Safety traps

**Iron law:** these compile, pass `go vet`, and fail in production. Check them on every diff that touches slices, maps, interfaces, or numeric widths.

- **A typed nil in an interface is not `nil`.** `var h *MyHandler; return h` from a function returning `http.Handler` yields a non-nil interface - the type descriptor is set. Return a literal `nil` for the nil case. Same trap for `error`: never declare a concrete `*MyError` return type; return `error`.
- **Writing to a nil map panics.** Reads, `len`, and `range` are fine, so the bug hides until the first write. Lazy-init inside the method: `if r.items == nil { r.items = make(map[string]int) }`.
- **`append` may alias.** If capacity allows, `b := append(a, x)` shares `a`'s backing array and writes through it. Force a fresh array with `append(a[:len(a):len(a)], x)` or `slices.Clone(a)`.
- **Return defensive copies.** An exported method returning `c.hosts` lets callers mutate your internals. Return `slices.Clone(c.hosts)` / `maps.Clone(c.m)`.
- **Narrowing integer conversions wrap silently.** `int32(int64(3_000_000_000))` is `-1294967296`. Bounds-check against `math.MaxInt32` / `math.MinInt32` before converting anything derived from user input, a request body, or a DB column.
- **Never compare floats with `==`.** `0.1+0.2 != 0.3`. Compare `math.Abs(a-b) < epsilon`.
- **Integer division by zero panics** (float division yields plus or minus Inf, or NaN). Guard the divisor and return an error.
- **Bare type assertions panic.** Always `v, ok := x.(T)`.

## Naming

- Variables: short in narrow scope (`i`, `r`, `tt`), descriptive in wide scope.
- Functions: verb for actions (`Process`), noun for getters — **no `Get` prefix**.
- Receivers: 1-2 letter abbreviation of the type (`s *Server`, `c *Client`). Never `this` or `self`.
- MixedCaps always; never underscores. Acronyms are all-caps or all-lower: `HTTPServer`, `httpServer` — not `HttpServer`.
- Packages: short, lowercase, singular noun. Avoid `util`, `helpers`, `common`, `misc`. No stutter — `user.Name()`, not `user.UserName()`.

## Testing

- Table-driven tests for any function with branching behavior. Use `t.Run` for sub-cases.
- `t.Helper()` in test helpers so failures point at the caller.
- `t.Cleanup()` for teardown; `t.TempDir()` for filesystem state.
- `t.Parallel()` when tests are independent (top-level + per-subtest).
- Match the project's existing assertion style (stdlib comparisons, `testify`, `go-cmp`). Don't introduce a new one.
- Assert all observable side effects, not just the primary return value.
- Buffered channels in concurrent test helpers to prevent goroutine leaks on failure.
- `-race` must pass. No `time.Sleep` for synchronization — use channels or test deadlines.
- No `defer` inside loops where the deferred call should run per-iteration; use a closure or refactor.

## Logging

- `log/slog` with structured fields as bare key-value pairs: `slog.Info("git sync: periodic pull started", "interval", s.interval)`. Never typed attributes (`slog.String`, `slog.Int`, `slog.Any`) - there are none in this codebase. No `fmt.Println` in production code.

## Quick red flags

| Red flag                                     | Why it's wrong                                                  |
| -------------------------------------------- | --------------------------------------------------------------- |
| `return err` without context                 | Caller can't tell where it came from                            |
| `interface{}` / `any` in a signature         | Loses type safety; use generics or concrete types               |
| Interface with 5+ methods                    | Hard to mock, weak abstraction — split                          |
| `panic()` for recoverable error              | Return an error                                                 |
| `go func()` with no shutdown path            | Resource leak                                                   |
| `strings.Contains` for control flow          | Use `==` for exact matches; Contains is for search              |
| `time.After` in a loop                       | Leaks timers — use `time.NewTimer` + `Stop()`                   |
| Unbounded map with no eviction               | Memory leak — add cap or use LRU                                |
| Repeated string literals for state/status    | Use typed constants                                             |
| `time.Sleep` for synchronization             | Use channels / context                                          |
| Package named `util`, `common`, `helpers`    | Generic name hides purpose                                      |
| `Get` prefix on a method                     | Convention is noun-only for getters                             |
| `this` / `self` receiver name                | Use 1-2 letter type abbreviation                                |
| `init()` doing I/O or starting goroutines    | Uncontrolled side effects on import                             |
| Bare `v := x.(T)` assertion                  | Panics on mismatch - use `v, ok := x.(T)`                       |
| Returning a typed nil pointer as an interface | Interface is non-nil - return a literal `nil`                  |
| Exported method returning an internal slice/map | Caller mutates your state - return a clone                   |
| `int32(v)` narrowing with no bounds check    | Silent wraparound on large values                               |
| Float compared with `==`                     | IEEE 754 is inexact - compare against an epsilon                |
