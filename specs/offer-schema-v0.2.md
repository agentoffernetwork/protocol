# Offer Schema v0.2 — Conversion Goals and Card Display

**Version:** 0.2 (stable, canonical)  
**Runtime support:** `not_available` until SVC-PLATFORM WS-15-S4 passes its runtime contract tests  
**Status:** formal public contract

This document defines the complete Offer payload for Protocol v0.2. It is an
independent snapshot of the v0.1 Offer surface plus the formal `goals[]`
conversion model and optional card display fields. Validate wire payloads with
[`offer-schema-v0.2.json`](../../schema/json-schema/offer-schema-v0.2.json),
then run the versioned semantic validator for event uniqueness, positive
decimal values, and display-template token rules.

## Complete Offer shape

The v0.2 object retains the v0.1 identity, content, entity, action, material,
targeting, source, extra, and attribution-rule fields with their existing
constraints. `version` is the literal string `"2.0"`. The top-level goals
field is REQUIRED, contains at least one item, has no protocol-defined upper
bound, and is the only canonical conversion pricing surface.

The top-level object is closed except for the explicit `extra` extension
object. The conversion rule retains its v0.1 attribution-window,
attribution-model, deduplication, and minimum-threshold fields. Its removed
compatibility list is replaced by the exact set of `goals[].event` values.

## Card display fields

The following OPTIONAL fields extend `offer_info` and `material[]` for
consumer-facing cards. They do not change the required identity, entity, action,
targeting, or goals semantics.

### Rating

`offer_info.rating` is an OPTIONAL closed object with REQUIRED `value` and
OPTIONAL `count` and `source`.

| field | rules |
|---|---|
| `value` | number from `0` to `5`, `multipleOf: 0.1` |
| `count` | non-negative integer; no protocol-defined upper bound |
| `source` | open string, may be empty, max 100 characters |

This field is a user-facing display rating. It is not an internal ranking,
confidence, or relevance score.

### Commercial display details

`offer_info.commercial.price.unit` is OPTIONAL and uses
`one_time | night | day | week | month | year`. A missing value is interpreted
as `one_time` for display semantics but MUST NOT be materialized during
round-trip. Consumers are responsible for locale-specific presentation such as
per-night or per-month copy.

`offer_info.commercial.fulfillment_note` is an OPTIONAL display-ready
transaction or fulfillment note, max 80 characters. Use it for one concise
condition such as cancellation, shipping, payment, or eligibility. It should
not duplicate structured `offer_info.properties[]` content. Put primary card
highlights such as "No annual fee", "Worldwide", "14 days", or "2% cashback" in
`offer_info.properties[]`; use `fulfillment_note` only for supporting terms or
fulfillment copy that is not meant to compete with the main highlights.

### Structured properties

`offer_info.properties` is an OPTIONAL ordered array with at most six items.
Each item is a closed object with REQUIRED `type` and `value`, plus OPTIONAL
`unit` and `display_pattern`.

| field | rules |
|---|---|
| `type` | open semantic key, pattern `^[a-z][a-z0-9]*(?:_[a-z0-9]+)*$`, max 64 |
| `value` | string, number, or boolean; string max 100 |
| `unit` | open string, max 32 |
| `display_pattern` | open plain-text template, max 120 |

`display_pattern` is not an enum and is not the semantic standardization field.
It is display copy. The semantic key is `type`: schema validates format only,
while documentation may list AON recommended property types for guidance. Such
a registry is not a JSON Schema enum, and unknown `type` values remain valid.

Recommended property types are guidance for producer consistency and consumer
native rendering. They are not a closed enum:

| type | intended meaning |
|---|---|
| `coverage` | country, region, availability, or service coverage |
| `fee` | explicit fee or fee waiver |
| `discount` | percentage or amount discount |
| `cashback` | cashback rate or amount |
| `reward` | points, miles, credits, or bonus value |
| `duration` | trial, subscription, validity, or stay duration |
| `requirement` | concise user requirement or eligibility highlight |
| `availability` | stock, booking, access, or timing highlight |

The template grammar recognizes only exact same-item placeholders:
`${type}`, `${value}`, and `${unit}`. Tokens may repeat or reorder. Ordinary
`$`, `{`, and `}` characters are text. Any unclosed `${`, spaced token such as
`${ value }`, unknown token, property access, or expression MUST be rejected by
the semantic validator. Substitution output is plain text only; consumers MUST
escape it and MUST NOT execute HTML or Markdown.

Consumer fallback rules:

- Known `type`: the consumer MAY use native or localized rendering.
- Unknown `type` with valid `display_pattern`: the consumer MAY render the
  plain-text template.
- Unknown `type` without `display_pattern`: the consumer MUST hide the item and
  MUST NOT guess display copy from `type`, `value`, or `unit`.

For localization, `display_pattern` may be translated, but `type`, `value`, and
`unit` are not automatically translated. The translated template must preserve
the same placeholder multiset as the source template; otherwise the translation
is discarded and the source template is used.

### Material accessibility

`material[].tag` remains an open string. The standard purposes are
`hero`, `banner`, `logo`, and `thumbnail`; custom tags remain valid. For
`tag=hero`, producers should provide a wide creative with aspect ratio at least
2.5:1. This is guidance, not a JSON Schema image-dimension blocker.

`material[].alt_text` is OPTIONAL, may be empty, and is limited to 200
characters. It is accessible replacement text for the creative asset: screen
readers, loading fallback, and localization. It is not a visible caption,
highlight, or promotional copy.

### Recommendation reason

`offer_info.recommendation_reason` remains an OPTIONAL string, max 1000
characters. Its v0.2 semantics are user-readable: partners provide explanatory
copy that consumers may choose to display. New examples and documentation must
not use internal metrics or operational-only notes as this field's content.

## Goals

Each goal contains REQUIRED `event` and `pricing`, plus OPTIONAL `description`.
`event` matches `^[a-z0-9][a-z0-9_.-]{0,63}$`; values are compared exactly and
must be unique within one Offer. No trim or canonicalization is applied.

`pricing` is a closed discriminated union:

| model | required fields | rules |
|---|---|---|
| `cpa` | `amount`, `currency` | exact positive decimal, up to 12 integer and 6 fractional digits; three-letter uppercase currency code |
| `cps` | `rate` | exact positive decimal, up to 4 fractional digits, at most 100; no currency |

`description` is advisory, may be empty, and is limited to 500 characters. It
does not affect matching, billing, settlement, or recognition.

## Version and runtime boundary

`AON-Protocol-Version: 0.2` selects this complete payload while the HTTP
`/v1/` shell remains unchanged. An omitted protocol header resolves to v0.1.
An explicitly unsupported or unknown version fails; it must not silently
downgrade. This formal source records a stable contract, not hosted runtime
availability. Runtime support is gated by SVC-PLATFORM WS-15-S4.

## Conversion-goal migration (non-normative)

Map fixed price to a `cpa` goal with the same amount and currency; map revenue
share to `cps.rate` without a currency. Hybrid pricing has no direct mapping
and requires manual reconstruction into one or more goals. The accepted event
set is now the `goals[].event` set. Serving-layer internal compatibility is a
follow-up and is not a public v0.2 field.

## Related sources

- [`offer-schema-v0.2.json`](../../schema/json-schema/offer-schema-v0.2.json)
- [`offer-v0.2.types.ts`](../../schema/types/offer-v0.2.types.ts)
- [`offer-response-v0.2.json`](../../examples/http/offer-response-v0.2.json)
- [`RFC-0002`](../../rfcs/rfcs/RFC-0002-conversion-goals-v0.2-formal.md)
- [`RFC-0003`](../../rfcs/rfcs/RFC-0003-offer-v0-2-card-display-fields.md)
