---
name: verification-before-completion
description: Use before claiming work is complete, fixed, or passing — requires running verification commands and confirming output before any success claim, before calling complete_task, and before transitioning a card to review or done
---

# Verification Before Completion

## Overview

Claiming work is complete without verification is dishonesty, not efficiency.

**Core principle:** Evidence before claims, always.

**Violating the letter of this rule is violating the spirit of this rule.**

## The Iron Law

```
NO COMPLETION CLAIMS WITHOUT FRESH VERIFICATION EVIDENCE
```

If you haven't run the verification command in this message, you cannot claim it passes.

## The Gate Function

```
BEFORE claiming any status, expressing satisfaction,
calling complete_task, or transitioning a card to review/done:

1. IDENTIFY: What command proves this claim?
2. RUN: Execute the FULL command (fresh, complete)
3. READ: Full output, check exit code, count failures
4. VERIFY: Does output confirm the claim?
   - If NO: State actual status with evidence
   - If YES: State claim WITH evidence
5. ONLY THEN: Make the claim or transition the card

Skip any step = lying, not verifying
```

## Common Failures

| Claim | Requires | Not Sufficient |
|-------|----------|----------------|
| Tests pass | Test command output: 0 failures | Previous run, "should pass" |
| Linter clean | Linter output: 0 errors | Partial check, extrapolation |
| Build succeeds | Build command: exit 0 | Linter passing, logs look good |
| Bug fixed | Test original symptom: passes | Code changed, assumed fixed |
| Regression test works | Red-green cycle verified | Test passes once |
| Subagent completed | Card body or git diff shows changes | Subagent reports "success" |
| Requirements met | Line-by-line checklist against card body | Tests passing |

## Red Flags - STOP

- Using "should", "probably", "seems to"
- Expressing satisfaction before verification ("Great!", "Perfect!", "Done!", etc.)
- About to call `complete_task` or `transition_card` to review/done without verification
- Trusting subagent success reports
- Relying on partial verification
- Thinking "just this once"
- Tired and wanting work over
- **ANY wording implying success without having run verification**

## Rationalization Prevention

| Excuse | Reality |
|--------|---------|
| "Should work now" | RUN the verification |
| "I'm confident" | Confidence ≠ evidence |
| "Just this once" | No exceptions |
| "Linter passed" | Linter ≠ compiler |
| "Subagent said success" | Verify independently |
| "I'm tired" | Exhaustion ≠ excuse |
| "Partial check is enough" | Partial proves nothing |
| "Different words so rule doesn't apply" | Spirit over letter |

## Key Patterns

**Tests:**
```
✅ [Run test command] [See: 34/34 pass] "All tests pass"
❌ "Should pass now" / "Looks correct"
```

**Regression tests (TDD Red-Green):**
```
✅ Write → Run (pass) → Revert fix → Run (MUST FAIL) → Restore → Run (pass)
❌ "I've written a regression test" (without red-green verification)
```

**Build:**
```
✅ [Run build] [See: exit 0] "Build passes"
❌ "Linter passed" (linter doesn't check compilation)
```

**Requirements:**
```
✅ Re-read card body → Create checklist → Verify each → Report gaps or completion
❌ "Tests pass, work complete"
```

**Subagent delegation:**
```
✅ Subagent reports success → Check git diff and card body → Verify changes → Report actual state
❌ Trust subagent report
```

## When To Apply

**ALWAYS before:**
- ANY variation of success/completion claims
- ANY expression of satisfaction
- ANY positive statement about work state
- Calling `complete_task` MCP tool
- Calling `transition_card` to `review` or `done`
- Moving to the next subtask
- Delegating to subagents

**Rule applies to:**
- Exact phrases
- Paraphrases and synonyms
- Implications of success
- ANY communication suggesting completion/correctness

## Common verification commands

If the project has a `Makefile`, prefer `make test` / `make lint` / `make build`.

**Those three are the minimum, not the whole gate.** Read every target in the
`Makefile` and every job in the CI workflow before claiming completion, and read
the verification section of the repo's `AGENTS.md` / `CLAUDE.md`. A repo's
`make test` is frequently scoped to one language or one directory: in a polyglot
repo the second suite usually sits behind its own target that `make test` never
invokes and CI still blocks the PR on. Gates a green `make test` will not catch:

- a per-language suite (`make test-frontend`, `make test-e2e`) covering code
  outside the repo's primary language;
- a race or sanitizer run as a separate job (`make test-race`, `-race`,
  `-fsanitize`);
- a formatter check the linter does not report on;
- a dependency, license, or module-boundary gate (`make deps-gate`,
  `go mod tidy -diff`, `npm audit`).

Enumerate this repo's own list once, at the start of the card, and verify against
that list - not against these examples.

Where no Makefile answers the question, discover the command rather than
guessing a language default: check the CI workflow, then `AGENTS.md` /
`CLAUDE.md`, then any checked-in wrapper script. The table below is the last
resort, not the first move:

| Language | Test | Lint | Build |
|----------|------|------|-------|
| Go       | `go test ./...` | `golangci-lint run` | `go build ./...` |
| Node/TS  | `npm test` | `npm run lint` | `npm run build` |
| Python   | `pytest` | `ruff check` | `ty check` |
| Rust     | `cargo test` | `cargo clippy` | `cargo build` |

**Compiling is not working.** A clean build and a clean typecheck prove the code
is well-formed, not that the change does what the card asked. When the card
describes runtime behavior, the changed path has to actually execute - a test
that covers it, or the real command run once - before you claim it works.

## The Bottom Line

**No shortcuts for verification.**

Run the command. Read the output. THEN claim the result.

This is non-negotiable.
