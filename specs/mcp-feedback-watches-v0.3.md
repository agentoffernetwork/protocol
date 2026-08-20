# MCP feedback / watches v0.3

> **Canonical v0.3 control contract**

This is the shared control envelope for REST and MCP. It does not by itself
claim that a particular deployment is live. REST and MCP implementations must
use the same `protocol_version`, `target`, `operation`, user-action requirement,
idempotency behavior and error categories.

```json
{
  "protocol_version": "0.3",
  "target": { "kind": "offer", "id": "offer-uuid" },
  "operation": "feedback",
  "user_action": "explicit",
  "idempotency_key": "client-generated-key",
  "feedback": "not_interested"
}
```

Supported operations are `feedback`, `watch`, and `unwatch`. A Query
response does not expose `watch_status` in the first release. After a Query
returns an Offer, the service may create or renew a default watch for later
Offer changes; that server-side behavior does not require a separate `watch`
call or add a C-side status field. The control envelope is only for an explicit
feedback action, cancellation (`unwatch`), or restore (`watch`). Missing user
action maps to `user_action_required`.

`feedback` targets an `offer` and accepts `dismissed` or `not_interested`.
`watch` and `unwatch` target an `offer` or `category` and must not carry a
`feedback` value. Replaying the same `idempotency_key` with the same canonical
payload is idempotent; reusing it with a different payload is an idempotency
conflict owned by the runtime implementation.
