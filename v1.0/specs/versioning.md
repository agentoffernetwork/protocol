# Version Namespace and Promotion Policy

## Stable v1.0

`AON-Protocol-Version: 1.0` selects the stable v1.0 Query contract. A successful
response echoes the same header, includes `protocol_version: "1.0"`, and varies
shared caches on `AON-Protocol-Version`. The Offer payload carries
`version: "3.0"`; that is the Offer document-model lineage and is intentionally
not the transport protocol token.

The header is mandatory. Omitted values and every value other than exact
`1.0`, including `0.3`, `0.2`, ranges, and approximate values, fail closed with
`unsupported_protocol_version`. No other selector is accepted or silently
reinterpreted by this contract.

## Future protocol lines

Stable v1.0 does not reserve a future selector. A future protocol line must
define and publish its own selector, response metadata, schemas, semantic
rules, types, examples, transition policy, and protected release evidence
before any deployment may negotiate it. Until then, every selector other than
exact `1.0` remains unsupported.

Runtime, OpenAPI, MCP, and SDK projections may not claim an unpublished
protocol selector. Their implementation drift is tracked separately and does
not redefine this canonical policy.
