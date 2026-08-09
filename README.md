# Task skills

Task skills mounted read-only into ContextMatrix worker containers at
`/run/cm-skills`, passed to the worker as `CMX_TASK_SKILLS_DIR`.

Agents do not auto-discover them. The worker's Go harness injects a one-line
`- name: description` menu of the offered skills into its coder, fix, document,
checkpoint-revise and review-specialist prompts; the model loads a skill by
calling the `skill` tool with that name. Only the menu line costs context until
a skill is actually called.

## Layout

One directory per skill at the repo root. Directory name = skill name
(referenced in `card.skills` and `project.default_skills`). Flat - no nesting.

```
go-development/SKILL.md
typescript-react/SKILL.md
...
```

The whole clone is always mounted; `card.skills` / `project.default_skills`
only narrow which skills are offered in the menu.

## Description-writing convention

The description is the only thing the model sees before it decides to call
`skill` - it is the whole menu line. Anchor descriptions in **observable
activities and file types**, not subject areas.

| ✗ Topic-shaped (engages too eagerly) | ✓ Task-shaped (engages on real work)                                |
| ------------------------------------ | ------------------------------------------------------------------- |
| "Go programming guidance"            | "Use when implementing or modifying Go source files."               |
| "All things React"                   | "Use when writing or updating React/TypeScript components."         |
| "Documentation"                      | "Use when writing or updating documentation files (README, docs/)." |
| "Code review"                        | "Use when reviewing changes for correctness or security issues."    |

The agent worker never offers skills during planning - the `skill` tool is only
wired into its coder, fix, document, checkpoint-revise and review phases. The
unfiltered surface is chat: the chat backend offers every mounted skill in every
session with no per-session narrowing, so a topic-shaped description will pull a
coding skill into unrelated conversations. A task-shaped description anchors
engagement to agents actually editing matching files.

Where a skill's scope is easy to overshoot, say so in the description. A
`Do NOT use when ...` clause that hands the excluded case to a named sibling
skill is the only signal the model gets before it commits.

## SKILL.md format

YAML frontmatter + markdown body.

```markdown
---
name: <skill-name-matching-dir-name>
description: Use when <observable activity>...
---

You are a <role>.

## When working on <activity>:

- Concrete pattern 1
- Concrete pattern 2
```

Only `description` is read from the frontmatter - by CM's lister
(`internal/api/task_skills.go`) and by the harness `skill` tool
(`contextmatrix-harness/tools/skill.go`). Every other key is ignored, and the
file is handed to the model with its frontmatter intact, so extra keys are
wasted tokens. There is no per-skill tool gate: `allowed-tools` is not honored
anywhere.

Two hard constraints, both silent when broken:

- `description` must be a **single line**. A folded (`>`) or multi-line value
  parses as empty and the skill is dropped from the menu with no error.
- The directory name must match `^[a-z0-9][a-z0-9._-]*$`. It is the callable
  name; the frontmatter `name` is not used for lookup.

## Supporting files

A skill directory may hold reference files alongside `SKILL.md`. They are loaded
on demand and cost nothing until asked for:

```
skill(skill="test-driven-development", file="testing-anti-patterns.md")
```

The `read` tool cannot reach them - it is jailed to the card's workspace, and
skills are mounted outside it. Any pointer to a supporting file must name the
`skill` tool call, or the file is unreachable in practice.

## Editing

This is your repo. Edit, add, remove as needed.

Point your ContextMatrix server at it with the `task_skills` block:

```yaml
task_skills:
  dir: /var/lib/contextmatrix/task-skills # local checkout on the CM host
  git_clone_on_empty: true
  git_remote_url: https://git.example.com/you/contextmatrix-task-skills # https only
```

Push to that remote to publish a change. CM fast-forwards its checkout once at
server startup and serves the resulting `{git_remote_url, ref}` pointer to the
agent and chat backends, which shallow-clone it and cache it for the lifetime of
the backend process. So a pushed change lands after CM restarts (or you pull its
checkout by hand) and the backend process restarts - not on the next trigger.

Verify a new skill with `GET /api/task-skills` and by seeing its name in a
worker's skill menu.

## Not task skills

Two other skill systems exist; don't confuse them with this repo.

- **Workflow skills** live in the contextmatrix repo under `workflow-skills/`
  and drive _what_ the agent does. Adding one needs both the markdown file and a
  `skillBuilders` entry in `internal/mcp/prompts.go` - `get_skill` returns
  `unknown skill` for any name absent from that map.
- **Interactive-session skills** (skill authoring, brainstorming, planning) run
  in Claude Code, where subagents exist. They do not belong here: a worker's
  model-callable tools are `read`, `edit`, `write`, `grep`, `glob`, `git`,
  `bash`, `finish` and `skill`, and nothing else.
