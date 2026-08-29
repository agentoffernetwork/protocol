# Offer Schema v0.3

> **Final canonical v0.3 contract**

The public Agent-facing carrier is
[`offer-schema-v0.3.json`](https://github.com/agentoffernetwork/schema/blob/main/v0.3/json-schema/offer-schema.json).
It is distinct from the Partner Offer artifact and the internal operator policy.
The representation boundaries in this document are normative.
The normative meaning of every Offer and Query response property is in
[Offer and Query Response Field Semantics v0.3](offer-field-semantics.md);
the corresponding JSON Schema `description` annotations are authoritative.

## Artifact boundary

| Artifact | Audience | Carrier | Contents |
| --- | --- | --- | --- |
| Public Offer | Agent and end user | `offer-schema-v0.3.json` | User-visible Offer, public goals and `goals[].pricing` |
| Partner Offer | Partner | `offer-partner-schema-v0.3.json` | Stable `source_offer_id`, Partner-authored public content, and Partner-only `targeting` and `conversion_rule`; no AON inventory identity, dispatch identity, or match reason |
| Internal Offer policy | Operator | `offer-internal-policy-v0.3.json` | `status`, `audit_status`, `priority`, provider identity, eligibility, freshness, affiliate and commission policy |

`targeting` and `conversion_rule` are rejected by the closed public Offer.
`offer_id`, `offer_instance_id`, and `match_reason` are rejected by the closed
Partner Offer: they are AON-authored public response projection fields, not
Partner supply fields. The Partner carrier requires stable `source_offer_id`;
AON resolves it within the configured Partner identity namespace and never
serializes it as canonical `offer_id`.
`offer_info.status`, `offer_info.audit_status`, and `offer_info.priority` are
internal policy keys and are rejected by both public and Partner Offer schemas.
They are never silently serialized into the Agent response.

## Public field groups

| Group | Meaning |
| --- | --- |
| `offer_info` | Display, category, commercial and descriptive content; `commercial.price` is public when supplied |
| `entity` | The merchant, brand, provider or publisher responsible for the offer |
| `listing_source` | Explicit observed platform/site metadata, never inferred from an action URL |
| `action` | The destination and operation offered to the user |
| `material` / `claims` | User-facing creative and bounded advertiser statements, not AON endorsement |
| `goals` | Public conversion event identity and public pricing declaration |
| `match_reason` | AON-generated, per-request match explanation controlled by `thinking_mode` |

`offer_id` is the AON-issued, globally unique inventory identity.
`offer_instance_id` identifies one public-response dispatch and must be
propagated unchanged through click and conversion attribution. It is generated
fresh for a later dispatch of the same Offer.

`listing_source` is supplemental source metadata, not a replacement for
`entity`. When present, `listing_source.logo` and `entity.logo` must be absolute
HTTPS URIs without userinfo. `entity.website` and every `material[].url` follow
the same resource rule. `action.payload.url` uses the type-aware safe URI policy
defined in the field-semantics reference; executable `javascript`, `data`,
`vbscript`, and `file` schemes are always invalid. `listing_source.logo` is never inferred from `entity.logo`,
`action.payload.url`, or `material[]`; when it is absent, consumers use
`listing_source.name` as the display and accessibility fallback.
`listing_source.url` remains outside the closed public object.

## Recommendation semantics

`offer_info.recommendation_reason` is Partner/provider-authored static Offer
copy. It may be shown to the user but is not an AON match explanation, ranking
input, or a `thinking_mode` control surface. `match_reason` is AON-authored,
per-request user-facing match copy. It is omitted when
`response_options.thinking_mode` is `false` and must not expose internal
reasoning or private ranking data. The canonical vectors define both fields in
[`recommendation-semantics.json`](https://github.com/agentoffernetwork/schema/blob/main/v0.3/fixtures/recommendation-semantics.json).

## Strict final-v0.3 conformance

After JSON Schema validation, canonical conformance applies the v0.3 semantic
validator. It requires the final-v0.3 language-tag profile, unique Goal events,
strictly positive CPA amounts and CPS rates, ordered `start_at`/`expire_at`, safe
resource and action URIs, valid display-pattern tokens, decimal price grammar,
and Taxonomy v1 membership/branch correctness. Partner payloads additionally
reject AON projection fields, empty targeting dimensions, empty conversion
rules, and non-positive minimum amounts. See the
[Producer migration and rejection guide](offer-migration.md).

## Goal commission basis

`offer_info.commercial.price` is the user-visible Offer price.
`goals[].pricing` is the gross commission basis the Partner declares payable to
AON for an approved, attributed Goal event. CPA is a fixed positive amount and
currency. CPS is a positive percentage, no greater than 100, applied to the
gross conversion amount and currency reported by the Partner. Neither branch is
the developer's net payout or a final settlement record; fees, developer share,
refunds, holds, disputes, and adjustments remain settlement lifecycle data.

The current contract deliberately does not define `decision_factors`. It uses
`engagement.refinements` for narrowing the current intent and
`engagement.followup_topics` for adjacent exploration.
