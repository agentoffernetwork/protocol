# MCP tools

> **Current stable tool contract**

This list registers four MCP tool names and their boundaries against the
current Query and control envelopes. A deployment claims availability only
after its owner provides the required conformance evidence. Deployments may
also expose additional tools through `tools/list`; those extensions are not a
portable Protocol v1.0 availability guarantee.

| Tool | Purpose | Contract source | Status |
| --- | --- | --- | --- |
| `aon_search_offers` | Match Offers to the user's current structured intent and return guidance | [Query API](query-api.md) + [Offer schema](offer-schema.md) | Contract |
| `aon_resolve_category` | Resolve a category reference before querying; returns category metadata, not decision factors | Deployment extension | Deployment-defined |
| `aon_submit_feedback` | Submit an explicit user feedback action for an Offer | [Feedback and watches](mcp-feedback-watches.md) | Contract; deployment-owned |
| `aon_manage_watch` | Explicitly restore or cancel the default Offer-change watch | [Feedback and watches](mcp-feedback-watches.md) | Contract; deployment-owned |

## Deployment extensions

Extensions must be discovered from the connected deployment's `tools/list`
response before use. The Hosted AON MCP deployment currently exposes the
following extension in addition to the four names above:

| Tool | Purpose | Contract source | Status |
| --- | --- | --- | --- |
| `aon_get_category_schema` | Return category decision factors for a clarification step before searching | Hosted `tools/list` schema | Optional Hosted extension |

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
- `aon_resolve_category` returns category metadata and candidate ids; it does
  not return decision factors.
- The optional `aon_get_category_schema` extension may return decision factors
  in its own result for clarification. `decision_factors` is not a field in the
  current Query request or response, and generic `next_actions` is not a
  current MCP contract field.
