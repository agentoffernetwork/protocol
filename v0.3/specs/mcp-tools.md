# MCP tools

> **Current stable tool contract**

This list aligns MCP names with the current Query and control envelopes. A
deployment claims availability only after its owner provides the required
conformance evidence.

| Tool | Purpose | Contract source | Status |
| --- | --- | --- | --- |
| `aon_search_offers` | Match Offers to the user's current structured intent and return guidance | [Query API](query-api.md) + [Offer schema](offer-schema.md) | Contract |
| `aon_resolve_category` | Resolve a category reference before querying; returns category metadata, not decision factors | Deployment extension | Deployment-defined |
| `aon_submit_feedback` | Submit an explicit user feedback action for an Offer | [Feedback and watches](mcp-feedback-watches.md) | Contract; deployment-owned |
| `aon_manage_watch` | Explicitly restore or cancel the default Offer-change watch | [Feedback and watches](mcp-feedback-watches.md) | Contract; deployment-owned |

## Shared rules

- `aon_search_offers` uses `response_options.thinking_mode`; it defaults to
  enabled and suppresses `offers[].match_reason` when disabled.
- Search input is the current intent plus a small allowed structured context;
  MCP must not require or transmit the entire chat transcript as a protocol
  field.
- Query may establish or renew the default watch after returning an Offer; this
  does not create an MCP call or expose `watch_status`.
- `aon_submit_feedback` and `aon_manage_watch` require a clear user action for
  feedback, cancellation, or restore. Inferred intent or simply displaying an
  Offer does not create one of those explicit control operations.
- Query responses do not expose first-release `watch_status`.
- `decision_factors` and generic `next_actions` are not MCP fields in the current contract.
