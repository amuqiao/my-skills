# Documentation Templates

Use these templates only when the repository does not already provide a stronger local format. Keep them short and adapt names to local conventions.

## API Route Contract

```markdown
# <Route / Capability Name>

## Purpose

One paragraph explaining what caller-visible job or request this endpoint performs.

## Method / Path

- `<METHOD> <PATH>`

## Request

List only caller-controlled fields and validation rules. Link shared schemas instead of copying them.

## Response

Describe accepted, processing, success, and failure shapes if the API is asynchronous. Link shared envelope/status/error specs.

## Errors

List stable external error codes or reasons. Link the canonical shared error table.

## Compatibility

State route-specific overrides and versioning rules. If a route-specific rule overrides a common spec, say so explicitly.
```

## Current Capability Page

````markdown
# <capability>

## Purpose

What this implemented capability does now.

## Current Behavior

- Current execution mode, queue/broker, provider, persistence, callback, artifact, and partial-success facts.

## Public Contract

External protocol: [`<route-doc>`](../api/...)

## Runtime Path

```text
api/...
  -> service/...
  -> queue/outbox/...
  -> worker/...
  -> provider/...
  -> repository/state update
  -> callback/artifact delivery
```

## Verification

- Contract tests:
- Worker/service tests:
- Smoke or integration checks:
````

## Current Runtime Flow

````markdown
# Runtime Flow

## Scope

This document describes implemented runtime behavior only.

## Flow

```text
request
  -> validation/auth
  -> persistence
  -> queue/outbox
  -> worker
  -> provider/integration
  -> terminal state
  -> callback/polling
```

## State Authority

Name the authoritative store for state transitions and recovery.

## Failure Semantics

Separate request failure, job failure, callback failure, and recovery behavior.

## Verification

List the tests or smoke commands that prove the flow.
````

## Active Plan

```markdown
# <Plan Name>

## Current Baseline

- Already implemented facts.

## Remaining Gaps

- Missing behavior, drift, or operational risk.

## Planned Work

- Smallest coherent work items.

## Acceptance

- Evidence required to close or supersede the plan.
```

## Drift Checklist

Use this as a review checklist, not as a mandatory file format:

```text
[ ] Agent entrypoints point to the correct current/API/plan docs.
[ ] Route docs match implemented route paths and methods.
[ ] Request/response docs match schema/types and examples.
[ ] Shared statuses, states, and error codes have one canonical source.
[ ] Capability/current matrix matches route files, worker/task registry, and tests.
[ ] Planned items are not described as current facts.
[ ] Completed plans were moved to current facts or marked/deleted as superseded.
[ ] Generated OpenAPI or equivalent projection is checked when the repo treats it as contract evidence.
```
