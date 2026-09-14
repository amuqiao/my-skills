---
name: implementation-contract-plans
description: Use when Codex needs to organize, create, repair, or review documentation that separates current implementation facts, external API or capability contracts, and future plans. Trigger for API docs, route/capability contract docs, current architecture docs, roadmap/plan cleanup, code-vs-doc drift checks, or converting scattered agent notes into maintainable current/contract/plan documentation.
version: 0.1.0
---

# Implementation Contract Plans

Use this skill to keep three kinds of project memory separate:

```text
current = what is implemented now
contract = what callers may rely on
plans = what is not implemented yet but remains worth doing
```

The goal is not to force a specific directory layout. Use the repository's existing names and documentation rules. If the repo already uses `docs/current`, `docs/api`, and `docs/plans`, follow those names. If it uses different files, preserve the local structure and map the three responsibilities onto it.

Do not use this skill for pure implementation, pure bug fixes, or ordinary code review when no API/capability contract, current architecture fact, documentation map, or future plan needs to be created, repaired, or checked.

## Workflow

1. Read the local agent entrypoints first.

   Start with `AGENTS.md` or the repo's equivalent, then read the closest module-level agent file if one exists. Treat those files as routing rules for where facts, contracts, and plans belong.

2. Classify the requested change.

   Use this split:

   | Kind | Owns | Must not contain |
   | --- | --- | --- |
   | `current` | Implemented architecture, module boundaries, runtime flow, data model, capability path, verification baseline. | TODOs, future promises, historical migration logs. |
   | `contract` | External API paths, request/response shapes, status/error semantics, callback payloads, version and compatibility rules. | Internal runtime details that callers cannot rely on. |
   | `plans` | Active gaps, planned work, acceptance criteria. | Completed facts, stale history, speculative design that has no owner. |

3. Find canonical sources before editing.

   For API work, inspect route files, schema/types, registries if present, generated OpenAPI or equivalent projections if the repo treats them as contract evidence, and contract tests if they exist. For capability or worker work, inspect the route, worker, service, shared schemas, runtime registry if present, and relevant tests. Do not let a Markdown page claim behavior that code and tests do not support.

4. Update the smallest complete set.

   - Contract change: update the route/API contract and add or adjust contract tests when the repo has a contract-test practice.
   - Current fact change: update the current implementation document. Update the nearest agent file only when the change materially affects module boundaries, dependencies, commands, or documentation ownership.
   - Plan change: update the plan only while the work is not implemented.
   - Completed plan: move the accepted fact into current docs or mark/remove the plan as superseded.

5. Run a drift check.

   Check matrices and lists that commonly diverge in this repo: route docs, route files, operation registries, capability enums, worker/task registries, current capability summaries, tests, OpenAPI snapshots, and agent entrypoints. Skip surfaces the repo does not have.

6. Verify.

   Run the narrowest tests that prove the changed contract or current fact. If validation cannot run, state exactly which surface remains unverified.

## Writing Rules

- Keep `current`, `contract`, and `plans` separate even when the same feature touches all three.
- Write current docs in present tense and only for shipped behavior.
- Write plans as gaps and acceptance criteria, not as pseudo-current documentation.
- Avoid duplicate field tables. A capability/current page should link to the route/API contract for external fields.
- Preserve a single canonical location for shared schemas, statuses, error codes, callbacks, and state machines.
- If common API docs and route-specific docs conflict, prefer the route-specific contract only when the repository explicitly uses that rule; otherwise follow the local conflict policy.
- Do not create shadow docs such as "temporary API notes", "supplemental contract", or a second API matrix unless the repo's documentation map says to.
- Prefer short tables, runtime paths, and ownership boundaries over broad architectural essays.

## Plan Template

Use this structure for active future work:

```markdown
# <Plan Name>

## Current Baseline

- What is already implemented and verified.

## Remaining Gaps

- What is still missing or risky.

## Planned Work

- The smallest coherent work items.

## Acceptance

- Observable criteria that allow the plan to be moved into current facts or closed.
```

## Reference Templates

When creating or repairing documents, read `references/doc-templates.md` for compact templates covering:

- API route contract
- current capability page
- current runtime flow
- active plan
- drift checklist
