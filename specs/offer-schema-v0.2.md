# Offer Schema v0.2

**Contract status:** current normative source contract

**Runtime support:** `not_available` until downstream runtime and release gates pass

This specification defines the complete canonical Offer returned by Query and
OfferProvider. The structural source is
[`offer-schema-v0.2.json`](https://github.com/agentoffernetwork/schema/blob/main/json-schema/offer-schema-v0.2.json);
the TypeScript projection is
[`offer-v0.2.types.ts`](https://github.com/agentoffernetwork/schema/blob/main/types/offer-v0.2.types.ts).

<a id="canonical-shape"></a>
<a id="requirement-levels"></a>
<a id="closed-objects"></a>

## Required shape

A v0.2 Offer is a closed object with:

- `offer_id`: stable inventory Offer identity;
- `offer_instance_id`: identity of this served instance;
- `version`: exact value `"2.0"`;
- `offer_info`: title, AON Taxonomy category, and description;
- `entity`: business identity;
- `action`: closed web redirect or app deep-link branch;
- `goals`: at least one billing Goal.

Optional fields include `content_language`, `material`, `targeting`,
`conversion_rule`, and open extension object `extra`.

<a id="no-bid"></a>
<a id="conversion-rule"></a>

Public v0.2 has no top-level `bid`. `conversion_rule` contains attribution
windows, attribution/dedup policies, and optional minimum conversion amount; it
has no `accepted_types`.

<a id="goal-event-name"></a>
<a id="goal-event-uniqueness"></a>

## Goals

Every item in `goals[]` requires:

```json
{
  "event": "subscription",
  "pricing": {
    "model": "cpa",
    "amount": "24.00",
    "currency": "USD"
  }
}
```

`event` is the Goal's cross-message identity. It uses the shared
[`goal-event-name-v0.2.json`](https://github.com/agentoffernetwork/schema/blob/main/json-schema/goal-event-name-v0.2.json)
slug grammar and must be exact-string unique within one Offer. Postback
`event_name` carries this exact value without mapping to a closed conversion
type.

The uniqueness rule is semantic. BL-033 records duplicate same-pricing and
different-pricing vectors as expected BL-035 failures; it does not claim that
JSON Schema or TypeScript alone enforces uniqueness.

<a id="cps"></a>

### Pricing

Pricing is exactly one closed branch:

```json
{"model": "cpa", "amount": "24.00", "currency": "USD"}
```

or:

```json
{"model": "cps", "rate": "20"}
```

<a id="cpa"></a>

CPA `amount` is a strictly positive decimal string and `currency` is uppercase ISO
4217 shape. CPS `rate` is a JSON string expressing a percentage in the
inclusive range `0..100`, with at most four decimals:

| Value | Result |
|---|---|
| `"0"`, `"0.0000"` | valid |
| `"99.9999"` | valid |
| `"100"`, `"100.0000"` | valid |
| negative, over 100, five decimals | invalid |
| leading zero such as `"01"` | invalid |
| JSON number, `currency`, or `amount` in CPS branch | invalid |

<a id="boundary-conversion"></a>

An internal fraction such as `0.2` maps explicitly to public percentage
`"20"` at the adapter boundary. Consumers must not guess units from a wire
value. BL-034/BL-035 own runtime mapping and enforcement.

## Taxonomy

`offer_info.category.id` is the primary AON Taxonomy v1 id.
`secondary_category_ids[]` contains auxiliary ids:

- all ids must exist in the registry;
- a secondary must be in a different branch from the primary;
- secondaries must be pairwise cross-branch;
- Query `constraints.category_ids[]` uses OR plus exact-or-descendant subtree
  matching against primary and secondaries.

Schema validates shape; registry membership and branch relations are semantic.

## Targeting

`targeting[]` is optional. It filters offer selection; it is not a legal or
regulatory compliance gate.

<a id="targeting-truth-table"></a>

| Rule | v0.2 semantics |
|---|---|
| missing or empty `targeting` | unrestricted |
| multiple rules | inter-rule OR |
| dimensions in one rule | active dimensions AND |
| omitted field, empty rule, `geo:{}`, empty include/exclude, empty device/OS | inactive and unrestricted |
| include lists | values inside one dimension OR |
| exclude | evaluated first and wins |

### Geo

Geo include/exclude arrays contain only structured `{location_id}` objects.
Legacy country strings are invalid. A viewer matches an include when its
self-or-ancestor location chain intersects the rule. Active geo fails closed
if location context is missing, unknown, or insufficient to prove exclusion
does not apply.

### Age

`eligibility.min_age` matches when any verified viewer threshold is greater
than or equal to the minimum. A known insufficient value fails. Missing
verified age passes for compatibility and must not be interpreted as verified
legal eligibility.

<a id="os"></a>

### Language, device, and OS

Offer language is lowercase ISO 639-1 and compares with the primary subtag of a
valid viewer BCP 47 value. Known mismatches fail; missing or invalid viewer
language passes as unknown.

Offer device values are `mobile`, `desktop`, `tablet`, and `smart_tv`.
Offer OS values are `ios`, `android`, `windows`, and `macos`. `linux` is not a
valid Offer targeting value. Query `other`, missing, or unknown device/OS
passes as unknown and is never treated as a Linux match.

## Display fields

<a id="commercial-price"></a>

`offer_info` may include display rating, user-readable recommendation reason,
structured properties, price/unit, and fulfillment note. `material[]` may
carry accessible `alt_text`. These fields are presentation inputs; they do not
replace taxonomy, targeting, Goal pricing, or internal ranking.

When `offer_info.commercial.price` is present, both its decimal-string `amount`
and ISO 4217 `currency` are required. `unit` remains optional.

`offer_info.properties[].display_pattern` may only use the same-item
placeholders `${type}`, `${value}`, and `${unit}`. Template-token checking is
semantic.

## Protocol usage

Both Query and OfferProvider success schemas reference this Offer by `$ref`.
Neither channel may inline or redefine Offer fields. `AON-Protocol-Version:
0.2` selects the complete v0.2 payload while the `/v1/` HTTP shell remains
unchanged.

## Related sources

- [Canonical Offer example](https://github.com/agentoffernetwork/examples/blob/main/http/offer-response-v0.2.json)
- [Query API v0.2](./query-api.md)
- [OfferProvider API v0.2](./offer-provider-api.md)
- [Postback v0.2](./postback.md)
- [RFC-0002](../rfcs/RFC-0002-conversion-goals-v0-2-formal.md)
