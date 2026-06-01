---
sidebar_position: 6
---

# delete_artifact

Stop and remove a deployed artifact. This deletes the deployment, stored source code, and all associated resources.

## Input Schema

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `artifact_id` | string | Yes | ID of the artifact to delete |

## Example

```json
{
  "artifact_id": "a1b2c3d4"
}
```

## Response

```json
{
  "message": "artifact my-portfolio deleted"
}
```

## What Happens

1. **Deletes** the `VibedApp` CR (the controller's owner-reference reaps the bound `SandboxClaim`, which releases the pod back to the warm pool)
2. **Removes** the stored source tarball from the configured source store
3. **Removes** the artifact record from the store
4. **Records** a `delete` event in the [audit trail](../configuration/audit-log.md) and emits a `deleted` event on the EventBus
