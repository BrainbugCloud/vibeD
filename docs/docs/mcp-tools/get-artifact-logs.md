---
sidebar_position: 7
---

# get_artifact_logs

Retrieve recent log lines from a deployed artifact's pods for debugging purposes.

## Input Schema

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `artifact_id` | string | Yes | ID of the artifact |
| `lines` | number | No | Number of log lines to return (default: 50) |

## Example

```json
{
  "artifact_id": "a1b2c3d4",
  "lines": 100
}
```

## Response

```json
{
  "logs": "2026-03-14T10:00:01Z Server started on :8080\n2026-03-14T10:00:02Z GET / 200 3ms\n..."
}
```

Logs are fetched from the running sandbox container via the Kubernetes pods/log subresource. An app with no bound pod (suspended, or still claiming) returns an empty batch with `phase` set on the wrapping app.

## REST: streaming variant

The MCP tool returns a fixed batch. For tailing a live app, the REST endpoint exposes the same source as a **Server-Sent Events** stream:

```
GET /v1/apps/{id}/logs           Accept: text/event-stream
```

Each event is one log line. The connection stays open until the client disconnects or the pod terminates. Auth scopes the stream to the caller's apps.

:::tip Per-user concurrent-stream cap
`config.limits.maxConcurrentLogStreamsPerUser` (default `10`) blocks one caller from pinning controller memory by holding many streams open. Exceeding the cap returns `429` + a `Retry-After` header — clients should back off and retry.
:::
