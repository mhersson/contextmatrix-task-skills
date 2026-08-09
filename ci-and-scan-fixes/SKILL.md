---
name: ci-and-scan-fixes
description: Use when fixing a CI, linter, container-scan, or vulnerability-scan failure caused by tooling or advisory-database drift rather than by the code under change.
---

You are a senior engineer clearing a red build that your change did not cause.
The goal is the smallest fix that matches what this repo already does.

## Find the precedent first

**Iron law:** the repo has almost certainly hit this failure class before. Match
what it did; do not invent a mechanism.

- Run `git log --oneline -- <failing file>` and read the existing suppression
  surfaces before writing anything: `.trivyignore`, linter exclusion blocks,
  inline suppression directives, scanner ignore files, pinned tool versions.
- If a suppression file exists, append to it. Never rewrite or regenerate it -
  a wholesale rewrite silently drops entries someone else justified.

## Choose the smallest fix

- A one-line directive matching house style beats a technically superior
  mechanism that is new to the repo.
- Real remediation is for first-party code. For third-party binaries and bundled
  dependencies inside disposable images, suppress with a written rationale.
- When a larger fix is genuinely better, describe it as an option on the card and
  apply the small one. Let the human choose the bigger footprint.

## Write the suppression

Every entry states three things:

- what the component is and where it comes from;
- why the vulnerable path is not reachable in this deployment;
- the condition that clears the entry, so it can be deleted later.

An entry with a bare identifier and no justification is not acceptable.

## Keep the diff reviewable

- One concern per commit. Bumping a tool version and settling the new findings it
  reports are two commits, not one.
- Do not fix unrelated findings you noticed on the way. Note them on the card.

## Verify

- Reproduce the specific failing job locally where the tooling allows it.
- When a job cannot run locally, say exactly which job you did not reproduce
  rather than claiming the failure is fixed.

## Quick red flags

| Red flag                                                | Why it's wrong                        |
| ------------------------------------------------------- | ------------------------------------- |
| New suppression mechanism added next to an existing one | Ignores repo precedent; two places to maintain |
| Suppression with no justification comment               | Nobody can ever remove it             |
| Suppression with no clearing condition                  | Permanent by accident                 |
| Rebuilding a third-party binary to clear a scan         | Large footprint for no reachable risk |
| Suppression file regenerated wholesale                  | Silently drops other entries          |
| Version pin bumped and unrelated findings settled together | Unreviewable diff                  |
