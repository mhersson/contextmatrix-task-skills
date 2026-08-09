---
name: removing-code
description: Use when deleting a feature, field, config key, endpoint, or dependency, or when renaming a concept across a codebase - covers surface enumeration, already-persisted data, and skew tolerance.
---

You are a senior engineer running a removal. Removal is not the inverse of
addition: the code goes away in one commit, the data and the deployments it
already touched do not.

## Enumerate before editing

**Iron law:** find every spelling of the thing before the first edit.

- Grep each form separately: Go identifier, JSON/YAML key, TypeScript field, UI
  label, metric name, config key, CLI flag, doc prose, test fixture, prompt or
  skill text. `CamelCase`, `snake_case`, and the human-readable label are three
  different searches.
- List the surfaces it crosses and work the list in order: domain type ->
  persistence -> service -> HTTP/API -> tool schemas -> UI -> config template ->
  docs -> tests.
- Removal is not done while any surface still names it.

## Persisted data outlives the code

**Iron law:** records written before your change still carry the field. Decide
explicitly - tolerate-and-ignore, or migrate. Never leave it undecided.

- Before assuming a leftover key is harmless, check whether the decoder is
  strict (`KnownFields(true)`, `DisallowUnknownFields`, strict YAML). A strict
  decoder turns a stale key into a hard failure at load, not a warning - and the
  failure lands on the operator at startup, not on you in CI.
- When you choose tolerate-and-ignore, leave one pin test that feeds the removed
  field through the reader and asserts it is ignored. The fixture must keep the
  removed key. That fixture is the test.
- When you choose migrate, the migration ships in the same change as the
  removal, not after it.

## Renames

- Pick hard cutover or alias, and state the choice on the card. An alias is a
  second removal you owe later.
- Rename the wire key, the storage key, and the display string in the same
  change, or none of them. A half-applied rename makes old records unreadable.
- A rename is a removal plus an addition. Everything above applies.

## Shared modules and multiple repos

- Change the shared module first, tag it, then update each consumer. Check what
  version each consumer pins; a consumer may have to skip intermediate versions.
- A removal in a shared wire contract is a breaking change even when your own
  consumer still compiles.
- Never bump a consumer to a shared-module version that is not yet tagged.

## Finish

- After the code compiles and tests pass, grep the old name once more across
  docs, config templates, prompts, and UI strings. This pass catches what the
  compiler cannot.
- Delete the tests that only existed to cover the removed path. Do not leave
  them skipped.
- If the change is breaking, say so in the commit subject and name the migration
  in the body.

## Quick red flags

| Red flag                                          | Why it's wrong                        |
| ------------------------------------------------- | ------------------------------------- |
| Removed a field but no test feeds the old value   | Existing records break silently       |
| Stale key left in a config template               | Strict decoders fail at startup       |
| Rename applied to code but not to the wire key    | Old records become unreadable         |
| Alias added "just in case"                        | A second removal you now owe          |
| Consumer bumped before the shared module is tagged | Unbuildable intermediate state        |
| Only the compiler consulted about completeness    | Docs, prompts and UI strings are text |
| Skipped test left behind instead of deleted       | Permanent dead weight in the suite    |
