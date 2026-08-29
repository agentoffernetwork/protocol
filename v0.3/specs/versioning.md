# Version Namespace and Promotion Policy

## Final v0.3

`AON-Protocol-Version: 0.3` selects the final v0.3 Query contract. A successful
response echoes the same header, includes `protocol_version: "0.3"`, and varies
shared caches on `AON-Protocol-Version`. The Offer payload carries
`version: "3.0"`; that is the Offer document-model lineage and is intentionally
not the transport protocol token.

The header is mandatory. Omitted and unrecognized selector values, including
`1.0`, fail closed with `unsupported_protocol_version`. Explicit `0.2` selects
only the documented legacy compatibility line. It is not a default and must
never be reinterpreted as final v0.3.

## Future protocol lines

Final v0.3 does not reserve a future selector. A future protocol line must
define and publish its own selector, response metadata, schemas, semantic
rules, types, examples, compatibility policy, and protected release evidence
before any deployment may negotiate it. Until then, every selector other than
the final v0.3 or explicit legacy v0.2 selector remains unsupported.

Runtime, OpenAPI, MCP, and SDK projections may not claim an unpublished
protocol selector. Their implementation drift is tracked separately and does
not redefine this canonical policy.
