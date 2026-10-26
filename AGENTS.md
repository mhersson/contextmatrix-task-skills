# AGENTS.md - ContextMatrix Task Skills

Rules for editing this repository. What task skills are, how they reach
agents, and how to write one: [`README.md`](README.md).

## Skills

- Skills are prose instructions: never add tests for them. Judge each line by
  whether deleting it would change what an agent does.
- Keep every skill generic: no ContextMatrix-codebase facts.

## Documentation

- Documentation must always be true: every statement matches the code at
  HEAD, and a change that makes a doc line false updates it in the same
  commit.
- Document the current state - what exists now and why, not how it got here.
- Comments explain only non-obvious decisions, constraints, safety invariants,
  or workarounds. Do not narrate what code does or record change history; git
  holds history. Keep comments to one or two tight lines unless a longer
  explanation is genuinely necessary. Rewrite or delete if you find comments
  that don't follow this rule.
- Never use em-dashes; use hyphens (-).
- Never reference plan phases, task numbers, or private card IDs in doc comments.

## Commit discipline

- **Never commit to `main`.** Work on a feature branch; tasks of a plan or
  scope the user approved commit there without asking again. Never push
  without explicit user approval.
- No session links in commits or PRs.
