---
name: receiving-code-review
description: Use when implementing changes from code review feedback or after a card was rejected from review back to in_progress - requires technical rigor and verification, not performative agreement or blind implementation
---

# Code Review Reception

## Overview

Code review requires technical evaluation, not emotional performance.

**Core principle:** Verify before implementing. Ask before assuming. Technical correctness over social comfort.

## The Response Pattern

```
WHEN receiving code review feedback (from your task prompt, from the
`## Review Findings (Round <N>)` section of the parent card body, or from a
human reviewer's notes):

1. READ: Complete feedback without reacting
2. UNDERSTAND: Restate requirement in own words (or ask)
3. VERIFY: Check against codebase reality
4. EVALUATE: Technically sound for THIS codebase?
5. RESPOND: Technical acknowledgment or reasoned pushback
6. IMPLEMENT: One item at a time, test each
```

If - and only if - `add_log` is in your tool list, reviewer feedback lives in
the **parent card body** under the heading `## Review Findings (Round <N>)` -
one section per review round. Call
`get_card(card_id=<parent_id>)` and read the highest-numbered round; earlier
rounds show what was already raised. The card's activity log carries only the
reviewer's `review_completed` entry
(`head=<sha> recommendation=<approve|revise>`), not the findings text. If your
own card body has a `## Source` section, also `get_card` that source ID - it
holds the original change context the findings reference.

When `add_log` is not in your tool list (the worker harness has no board
tools), the findings arrive in your task prompt: read them there, and call none
of the board tools this skill names.

## Forbidden Responses

**NEVER:**
- "You're absolutely right!" (performative)
- "Great point!" / "Excellent feedback!" (performative)
- "Let me implement that now" (before verification)

**INSTEAD:**
- Restate the technical requirement
- Ask clarifying questions (in chat for HITL; for async, via an `add_log`
  comment if - and only if - `add_log` is in your tool list, else as Handling
  Unclear Feedback below says)
- Push back with technical reasoning if wrong
- Just start working (actions > words)

## Handling Unclear Feedback

```
IF any item is unclear:
  DO NOT guess and DO NOT implement the clear items in isolation
  HITL (a human is in the loop): ask in chat and wait
  Autonomous, if - and only if - `add_log` is in your tool list
    (you are a sub-agent): record the ambiguity with `add_log`,
    then follow execute-task Step 8 - `transition_card` to `blocked`
    and print TASK_BLOCKED with `needs_human: true`. Never stop silently:
    nothing reads add_log for answers during an autonomous run.
  Autonomous, `add_log` not in your tool list: nothing can answer and
    nothing reads a blocked token, so do not stop and do not print
    TASK_BLOCKED. Instead of guessing, choose the reading the code and the
    finding's own text best support, implement all items under that
    reading, and name each ambiguity and the reading chosen in the summary
    you finish with (the `finish` tool's commit message in the worker).

WHY: Items may be related. Partial understanding = wrong implementation.
```

**Example:**
```
Reviewer: "Fix items 1-6"
You understand 1,2,3,6. Unclear on 4,5.

❌ WRONG: Implement 1,2,3,6 now, ask about 4,5 later
✅ RIGHT: "I understand items 1,2,3,6. Need clarification on 4 and 5
          before proceeding." (in HITL, asked in chat; autonomous with
          `add_log` in your tool list, recorded via add_log then reported as
          TASK_BLOCKED with needs_human: true)
✅ RIGHT, autonomous without `add_log`: implement all six under the reading
          of 4 and 5 the code best supports, and name that reading in the
          summary you finish with.
```

## Source-Specific Handling

### From the human user (HITL)
- **Trusted** - implement after understanding.
- **Still ask** if scope unclear.
- **No performative agreement.**
- **Skip to action** or technical acknowledgment.

### From a reviewer subagent (autonomous)
In an autonomous run with `add_log` in your tool list you normally receive
findings twice over: the orchestrator copies each Critical/Important finding
verbatim into your fix subtask's body (with its acceptance criterion), and the
full synthesized set - including Minor and Nit tiers - sits in the parent card
body under `## Review Findings (Round <N>)`. Read both before acting. Without
`add_log`, read the findings in your task prompt. Treat them the same way you'd
treat any technical critique:

```
BEFORE implementing:
  1. Check: Technically correct for THIS codebase?
  2. Check: Breaks existing functionality?
  3. Check: Reason for current implementation?
  4. Check: Works on all target platforms/versions?
  5. Check: Did the reviewer have full context?

RECORD each case below via add_log if - and only if - `add_log` is in your
tool list. Otherwise state it, with the technical reasoning and file:line
evidence, in the summary you finish with; no one can answer, so never pause:
proceed with the option the evidence supports, or leave a conflicting
suggestion unimplemented, and say which.

IF suggestion seems wrong:
  Push back with technical reasoning via add_log
  Do NOT silently disregard - leave a paper trail.

IF can't easily verify:
  Say so via add_log: "Cannot verify [X] without [Y].
  Proceeding with [option]/Pausing for input."

IF conflicts with prior architectural decisions:
  Stop. Add a log entry flagging the conflict.
```

## YAGNI Check for "Professional" Features

```
IF reviewer suggests "implementing properly":
  grep codebase for actual usage

  IF unused: remove it and record "Endpoint isn't called. Removed (YAGNI)." -
    via add_log if it is in your tool list, else in the summary you finish with
  IF used: implement properly
```

## Confidence-Weighted Triage

Before implementing, triage each finding. Record two judgments per finding:

**Confidence (low / medium / high):**
Can I fix this without introducing a new defect? Have I read enough of the
surrounding code to apply the same cross-cutting concerns (redaction, schema
validation, accessibility spec, etc.) that govern the file?

**Decision (fix / defer / reject):**
- `fix` - Critical and Important findings, regardless of confidence. Low
  confidence is a signal to read more before fixing, not to defer.
- `fix` - Minor findings at high confidence.
- `defer` - Minor findings at low or medium confidence. Spawn a follow-up with
  `create_card(project=<this card's project>, title='Follow-up: <one-line
  finding summary>', body=<the finding's full Where/What/Why/Fix block>,
  type=<same type as the card under review>, priority='low',
  parent=<the parent of the card under review, or omit if it has none>)`, if -
  and only if - `add_log` is in your tool list.
  Do **not** parent the follow-up to the card under review - `create_card`
  overrides `type` to `subtask` when `parent` is set, and the orchestrator's
  next `get_ready_tasks` sweep would spawn it immediately instead of deferring
  it. Without `add_log` no follow-up card can be created: list each deferral,
  with the finding's file:line and your reason, in the summary you finish with.
  A speculative fix that introduces a new defect is worse than a tracked
  follow-up.
- `reject` - verification shows the finding is wrong. Record reasoning via
  `add_log` if - and only if - `add_log` is in your tool list, else in the
  summary you finish with (same pattern as the existing pushback section).

**From-scratch test (for Minor judgments):** if I were writing this code
from scratch knowing what I know now, would I write it this way?
- If **no** → `fix`.
- If **yes** → likely `defer` (style preference) or `reject` (wrong critique).

Make the triage decisions **before** any code change. If - and only if -
`add_log` is in your tool list, write them down with
`update_card(card_id=<your card_id>, agent_id=<your agent_id>,
upsert_section_heading='Revision Plan', upsert_section_content=<bullets>)`.
Never re-emit the whole body - the upsert leaves every other section,
including `## Review Findings`, untouched. One bullet per finding:

```
- <Severity> | <file:line> - <one-line summary> → **<decision>** (<confidence>; <optional note>).
```

This is the durable audit trail for a human reading the card. It is **not**
automatically visible to the next review pass - review-task receives subtask
summaries without bodies, and its injected parent body is filtered to the intro,
`## Plan`, `## Review Findings`, and `## Decisions`. So also record each
deferral in the activity log, which the next reviewer does read:
`add_log(card_id=<parent_id>, agent_id=<your agent_id>, action='finding_deferred',
message='<severity> <file:line> deferred: <reason> (follow-up <card-id>)')`.

When `add_log` is not in your tool list, skip this `update_card` and this
`add_log`: carry each defer and reject decision, with its file:line and
reason, into the summary you finish with.

## Implementation Order

```
FOR multi-item feedback:
  1. Clarify anything unclear FIRST
  2. Then implement in this order:
     - Blocking issues (breaks, security)
     - Simple fixes (typos, imports)
     - Complex fixes (refactoring, logic)
  3. Test each fix individually
  4. Verify no regressions before moving to next item
```

## When To Push Back

Push back when:
- Suggestion breaks existing functionality
- Reviewer lacks full context
- Violates YAGNI (unused feature)
- Technically incorrect for this stack
- Legacy/compatibility reasons exist
- Conflicts with prior architectural decisions

**How to push back:**
- Use technical reasoning, not defensiveness
- Ask specific questions
- Reference working tests/code (with file:line citations)
- Record in `add_log` so the next reviewer sees the chain of reasoning, if -
  and only if - `add_log` is in your tool list; otherwise state it in the
  summary you finish with

## Acknowledging Correct Feedback

When feedback IS correct:
```
✅ "Fixed. [Brief description of what changed]"
✅ "Good catch - [specific issue]. Fixed at [file:line]."
✅ [Just fix it and show in the code]

❌ "You're absolutely right!"
❌ "Great point!"
❌ "Thanks for catching that!"
❌ ANY gratitude expression
```

**Why no thanks:** Actions speak. Just fix it. The code itself shows you heard the feedback.

**If you catch yourself about to write "Thanks":** DELETE IT. State the fix instead.

## Gracefully Correcting Your Pushback

If you pushed back and were wrong:
```
✅ "You were right - I checked [X] and it does [Y]. Implementing now."
✅ "Verified this and you're correct. My initial understanding was wrong because [reason]. Fixing."

❌ Long apology
❌ Defending why you pushed back
❌ Over-explaining
```

State the correction factually and move on.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Performative agreement | State requirement or just act |
| Blind implementation | Verify against codebase first |
| Batch without testing | One at a time, test each |
| Assuming reviewer is right | Check if breaks things |
| Avoiding pushback | Technical correctness > comfort |
| Partial implementation | Clarify all items first |
| Can't verify, proceed anyway | If - and only if - `add_log` is in your tool list, state limitation in `add_log` and ask for direction; otherwise state it, and the option you took, in the summary you finish with |

## Real Examples

**Performative Agreement (Bad):**
```
Reviewer: "Remove legacy code"
❌ "You're absolutely right! Let me remove that..."
```

**Technical Verification (Good):**
```
Reviewer: "Remove legacy code"
✅ "Checking… build target is 10.15+, this API needs 13+. Need legacy
   for backward compat. Either fix the bundle ID (current is wrong) or
   drop pre-13 support - flagging."
```

**YAGNI (Good):**
```
Reviewer: "Implement proper metrics tracking with database, date filters, CSV export"
✅ "Grepped codebase - nothing calls this endpoint. Removing (YAGNI).
   Logged in card for reviewer."
```

## The Bottom Line

**Reviewer feedback = suggestions to evaluate, not orders to follow.**

Verify. Question. Then implement.

No performative agreement. Technical rigor always.
