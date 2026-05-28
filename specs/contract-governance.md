# Contract Governance

AgentOffer Protocol uses explicit contract governance to keep the protocol, schema, examples, runtime OpenAPI, docs, SDKs, and public websites aligned.

Protocol specifications and JSON Schema define wire-level canonical fields. Downstream public surfaces must follow those sources with an explicit `source_ref`; they should not redefine canonical fields independently.

## Lifecycle States

| State | Meaning |
| --- | --- |
| `canonical` | Current recommended public contract field |
| `deprecated` | Still present, but not recommended for new integrations |
| `compatibility` | Accepted or shown only for backwards compatibility |
| `internal` | Internal implementation or observability field |
| `historical` | Historical or migration context |
| `preview` | Early public preview, not stable |
| `draft` | Proposal-only field |
| `removed` | No longer part of the public contract |

## Public Follow Rules

Public surfaces that mention protocol/API fields should declare:

| Field | Meaning |
| --- | --- |
| `follows_contract` | Contract ID followed by the surface |
| `source_ref` | Upstream protocol source path and release/commit reference |
| `last_verified_at` | Last verification date |
| `known_exceptions` | Compatibility or historical exceptions |

Covered surfaces include the public protocol repository, schema repository, examples repository, AON protocol website, docs-site API Reference, runtime OpenAPI, SDK public types, and ChatGPT Action OpenAPI.

## First Governed Contract: `offers.query`

Canonical fields:

| Field | Meaning |
| --- | --- |
| `AON-Protocol-Version` | HTTP header that pins the AgentOffer Protocol payload contract version, for example `0.1` |
| `X-AON-TRACE-ID` | Hosted Query API HTTP response header for support diagnostics; not a JSON body field |
| `request_id` | Query/request correlation identifier |
| `offers` | Ranked Offer objects returned by the query |
| `offers[].offer_id` | Inventory-level stable Offer ID |
| `offers[].offer_instance_id` | Per-dispatch Offer instance ID |

Canonical agent-facing request constraints:

| Field | Meaning |
| --- | --- |
| `constraints.category_ids` | Structured AON Taxonomy v1 eligibility constraint |

Historical, compatibility, internal, or removed fields:

| Field | Status | Handling |
| --- | --- | --- |
| `query_id` | historical / removed | Use `request_id` |
| `trace_id` | internal / compatibility | Internal observability or historical provider envelope only; not `offers.query` public response canonical |
| `aon_trace_id` | internal / transport header only | AON runtime diagnostic id. Expose to Query callers only as the `X-AON-TRACE-ID` HTTP response header, never as a JSON body field. |
| `has_more` | historical / compatibility | Not current `offers.query` public response canonical unless runtime OpenAPI reintroduces pagination |
| `total` | historical / compatibility | Not current `offers.query` public response canonical unless runtime OpenAPI reintroduces pagination |
| `uuid` | removed | Use `offer_id` or `offer_instance_id` depending on semantics |
| `original_offer_id` | removed | Use `offer_id` |
| `source_offer_id` | removed | Removed from the public contract |
| `filter` | removed from Query API and OfferProvider API request bodies | Use root `constraints`. |
| `QueryFilter` | removed from agent-facing Query API | Use `QueryConstraints`. |
| `constraints.category_types` | removed from Taxonomy v1 Query and OfferProvider request bodies | Use `constraints.category_ids`. |
| `filter.status` | removed from Query API and OfferProvider API request bodies | Query API and OfferProvider dispatches return active eligible offers by default. |
| `filter.bid_models` | removed from agent-facing Query API | Bid model is supply/commercial selection logic, not a public client constraint. |
| `filter.currency` | removed from agent-facing Query API | Currency constraints are not public in this version. |
| `filter.min_bid_amount` | removed from agent-facing Query API | Bid constraints are not public in this version. |
| `filter.max_price_amount` | removed from agent-facing Query API | Price constraints are not public in this version. |
| `filter.brand` | removed from agent-facing Query API | Brand constraints are not public in this version. |
| `filter.country` | removed from agent-facing Query API | Country constraints are not public in this version. |

## Machine-Readable Manifest

The internal monorepo maintains a machine-readable manifest at:

```text
protocol/docs/contract-governance/contracts.json
```

Public releases should keep the protocol, schema, and examples repositories aligned with that manifest. Future drift guards should consume the manifest rather than scraping prose.

Two manifest keys are especially important for downstream checks:

| Manifest key | Purpose |
| --- | --- |
| `stale_field_denylist` | Fields that must not appear in active public integration paths without an allowed context |
| `compatibility_allowlist` | Historical, migration, compatibility, or internal contexts where otherwise stale field names may appear |
| `taxonomy_registry` | AON Taxonomy v1 source tree, migration mapping, and drift guard source refs |

## Release Checklist

Before publishing public protocol surfaces:

1. Update the contract manifest.
2. Confirm lifecycle state for every new or changed field.
3. Confirm downstream `source_ref` values.
4. Scan `stale_field_denylist`.
5. Review `compatibility_allowlist` context.
6. Confirm public GitHub protocol/schema/examples are aligned.
7. Register downstream follow-up work for known remaining drift.
8. Run the AON Taxonomy v1 drift guard when taxonomy ids or examples change.
