---
sidebar_position: 8
---

# Audit Trail

vibeD records mutating actions to an append-only audit trail so you can answer "who changed what, when" — a baseline enterprise-governance requirement. The trail is always on; what differs is whether it persists (see [Storage](#storage)).

## What's recorded

Each event captures the **actor** (authenticated user ID), **action**, **target** app, **outcome**, an optional **detail**, and a timestamp:

| Action     | Recorded when                                  | Outcomes                |
| ---------- | ---------------------------------------------- | ----------------------- |
| `deploy`   | a new app or redeploy (via API or MCP)         | `ok`, `denied`, `error` |
| `delete`   | an app is deleted (via API or MCP)             | `ok`, `error`           |
| `rollback` | an artifact is rolled back to a prior version  | `ok`, `error`           |
| `suspend`  | `POST /v1/apps/{id}/suspend` flips suspended on| `ok`, `error`           |
| `resume`   | `POST /v1/apps/{id}/resume` flips it off       | `ok`, `error`           |

`outcome=denied` is how a [quota](./quotas.md) rejection shows up; `error` carries the failure reason in `detail`.

## Querying it

Admins read the trail over the API (the `admin` role is required — non-admins get `403`):

```bash
curl -H "Authorization: Bearer $ADMIN_TOKEN" \
  "https://vibed.example.com/v1/audit?actor=alice&action=deploy&app=my-site&limit=100"
```

```jsonc
{ "events": [
  { "time": "2026-05-24T10:02:11Z", "actor": "alice", "action": "deploy", "target": "my-site", "outcome": "ok" },
  { "time": "2026-05-24T09:58:03Z", "actor": "bob",   "action": "delete", "target": "old-demo", "outcome": "ok" }
] }
```

All filters (`actor`, `action`, `app`, `limit`) are optional; results come back newest-first.

## Storage

- **SQLite backend** (`store.backend: sqlite`): events persist in the same database as everything else and survive restarts.
- **Memory / ConfigMap backends**: the trail is kept **in memory only** and is lost on restart (vibeD logs a warning at startup). Use SQLite for a durable audit trail.

Every recorded event also increments `vibed_audit_events_total{action,outcome}` and is mirrored to structured logs.

:::note Egress denials
Blocked outbound connections are logged separately by the egress proxy's authorizer (see [Egress Control](./egress-control.md)), not in this trail — they happen in a different process on the request hot path.
:::

## Fail-closed mode

For compliance contexts where an **untraceable mutation is worse than an unavailable API**, flip the recorder fail-closed:

```yaml
config:
  audit:
    failClosed: true
```

When set:

- A success-path audit write that fails (disk full, store unreachable) causes the API to return an error. The underlying action — the `VibedApp` is already created or deleted — won't be rolled back, but the caller sees the failure and knows to retry (deploys are idempotent) or alert. The Prometheus counter and the structured log line are still emitted.
- Pre-action audit failures (e.g. failing to persist the `denied` record for a quota-rejected deploy) are intentionally **swallowed** — the original cause (the quota error) already propagates to the caller, and double-erroring would mask it.

The default is `failClosed: false` so the install boots cleanly without a persistent store wired. Flip to `true` in production values alongside `store.backend: sqlite`.
