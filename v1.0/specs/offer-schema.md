# Offer Schema v1.0

> **Final canonical v1.0 contract**

The public Agent-facing carrier is
[`offer-schema-v1.0.json`](https://github.com/agentoffernetwork/schema/blob/main/v1.0/json-schema/offer-schema.json).
It is distinct from the Partner Offer artifact and the internal operator policy.
The representation boundaries in this document are normative.
The normative meaning of every Offer and Query response property is in
[Offer and Query Response Field Semantics v1.0](offer-field-semantics.md);
the corresponding JSON Schema `description` annotations are authoritative.

## Artifact boundary

| Artifact | Audience | Carrier | Contents |
| --- | --- | --- | --- |
| Public Offer | Canonical public response facts | `offer-schema-v1.0.json` | User-visible Offer, public goals, `goals[].pricing`, registered optional supply-profile facts, and AON-owned response presentation data |
| Partner Offer | Partner | `offer-partner-schema-v1.0.json` | Stable `source_offer_id`, Partner-authored public content, and Partner-only `targeting` and `conversion_rule`; no AON inventory identity, dispatch identity, or match reason |
| Query Generic Offer projection | Hosted Query and MCP consumer | `offer-query-generic-projection-v1.0.json` | Stable Generic Offer response shape; rejects supply-profile and observed-commercial extensions |
| Internal Offer policy | Operator | `offer-internal-policy-v1.0.json` | `status`, `audit_status`, `priority`, provider identity, eligibility, freshness, affiliate and commission policy |

`targeting` and `conversion_rule` are rejected by the closed public Offer.
`offer_id`, `offer_instance_id`, and `match_reason` are rejected by the closed
Partner Offer: they are AON-authored public response projection fields, not
Partner supply fields. The same boundary applies to
`offer_info.commercial.display_price`: it is AON-authored response presentation
data and is rejected by Partner Offer and OfferProvider success carriers. The
Partner carrier requires stable `source_offer_id`;
AON resolves it within the configured Partner identity namespace and never
serializes it as canonical `offer_id`.
`offer_info.status`, `offer_info.audit_status`, and `offer_info.priority` are
internal policy keys and are rejected by both public and Partner Offer schemas.
They are never silently serialized into the Agent response.

## Supply profile extensions

`offer_info.details` is an optional, closed supply-profile envelope. The stable
v1.0 registry at `offer-profile-registry-v1.0.json` recognizes only `flight`
and `hotel_rate`; each profile selects one closed `details.data` shape. Generic
Offers omit `details`. A producer must not use an unregistered profile name or
a free-form domain payload.

These fields are canonical supply facts for the Public Offer, Partner Offer,
and Provider success carriers. They do not extend the current Hosted Query or
MCP response: those surfaces bind
`offer-query-generic-projection-v1.0.json`, which rejects
`offer_info.details`, `offer_info.commercial.price.tax_status`, and
`offer_info.commercial.quote`. A future public runtime projection requires its
own certified rollout; it is not selected by a new header or Offer version.

## Response display price

`offer_info.commercial.display_price` is an optional, closed object allowed on
the Public Offer and Generic Query Offer carriers. It is owned by the AON Query
response projection and requires exactly a canonical non-negative decimal
string `amount` and a three-uppercase-ASCII-letter `currency`. It must not
contain `unit`, `tax_status`, `quote`, `fulfillment_note`, or another member.

When present, it requires `commercial.price`, uses a currency different from
the original price, and preserves zero-value consistency: zero source prices
have zero display prices, while positive source prices have positive display
prices. Only complete absence permits fallback to `price`; a present but
invalid value is a contract error.

Consumers form the presentation value by overlaying only `amount` and
`currency` onto `price`. Existing `price.unit` and available
`price.tax_status` remain source-price qualifiers. `display_price` is scoped to
the current response and is never checkout, settlement, or
transaction-authoritative. See
[RFC-0004](https://github.com/agentoffernetwork/rfcs/blob/main/rfcs/RFC-0004-offer-display-price.md)
and the field-semantics reference for the complete consumption, freshness, and
carrier rules.

## Public field groups

| Group | Meaning |
| --- | --- |
| `offer_info` | Display, category, commercial and descriptive content; `commercial.price` is the original price, optional `commercial.display_price` is response-scoped presentation data, and optional `details` carries a registered supply profile |
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
[`recommendation-semantics.json`](https://github.com/agentoffernetwork/schema/blob/main/v1.0/fixtures/recommendation-semantics.json).

## Strict stable-v1.0 conformance

After JSON Schema validation, canonical conformance applies the v1.0 semantic
validator. It requires the stable-v1.0 language-tag profile, registered and
internally consistent supply profiles when `details` is present, unique Goal events,
strictly positive CPA amounts and CPS rates, ordered `start_at`/`expire_at`, safe
resource and action URIs, valid display-pattern tokens, decimal price grammar,
display-price ownership and cross-field consistency,
and Taxonomy v1 membership/branch correctness. Partner payloads additionally
reject AON projection fields, empty targeting dimensions, empty conversion
rules, and non-positive minimum amounts. The executable rejection corpus is
published with the v1.0 semantic vectors and validators.

## Goal commission basis

`offer_info.commercial.price` is the original user-visible Offer price.
An optional `offer_info.commercial.display_price` changes only which amount and
currency a consumer presents for the current response; it never replaces the
source price for checkout, Goal commission, attribution, or settlement.
`goals[].pricing` is the gross commission basis the Partner declares payable to
AON for an approved, attributed Goal event. CPA is a fixed positive amount and
currency. CPS is a positive percentage, no greater than 100, applied to the
gross conversion amount and currency reported by the Partner. Neither branch is
the developer's net payout or a final settlement record; fees, developer share,
refunds, holds, disputes, and adjustments remain settlement lifecycle data.

The current contract deliberately does not define `decision_factors`. It uses
`engagement.refinements` for narrowing the current intent and
`engagement.followup_topics` for adjacent exploration.
