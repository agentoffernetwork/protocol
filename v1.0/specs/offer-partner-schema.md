# Partner Offer Schema v1.0

> **Final canonical Partner artifact**

The Partner Offer artifact uses `offer-partner-schema-v1.0.json`. It is the
normalized Partner supply/configuration carrier submitted before AON creates a
public Offer projection. It contains the Partner-only stable source identity
`source_offer_id`, Partner-authored public Offer content, and two Partner-only
configuration fields: `targeting` and `conversion_rule`.

The Partner artifact requires `source_offer_id`, which is opaque and stable in
the identity namespace configured for that integration. AON resolves
`(Partner, identity namespace, source_offer_id)` to its canonical `offer_id`.
The Partner artifact rejects AON-owned `offer_id`, `offer_instance_id`, and
`match_reason`; AON creates the latter two only while producing a public Agent
response. It is not an Agent response schema and not a replacement for
`offer-schema-v1.0.json`.

## Partner-only fields

| Field | Purpose | Strict stable-v1.0 rule |
| --- | --- | --- |
| `source_offer_id` | Stable Partner source identity | Required; opaque and stable within the configured integration identity namespace; never copied into public `offer_id` |
| `targeting` | Eligibility, geography, language, device and OS constraints | Rules are ORed; dimensions in one rule are ANDed; exclusion wins; unknown declared context fails that rule closed; all supplied arrays and groups are non-empty; every location id exists in the pinned AON Location Registry v1 snapshot |
| `conversion_rule` | Attribution windows, model, deduplication and minimum amount | When supplied it is non-empty; windows are `0..8760`; omitted members use canonical defaults; `minimum_amount` is strictly positive |

Public `goals[].pricing` remains public in both carriers. Do not move pricing,
action data, title, description, entity, or listing-source metadata into
Partner-only configuration merely because a Partner authored it.

`offer_info.status`, `offer_info.audit_status`, and `offer_info.priority` are
operator-only policy keys. The Partner artifact rejects them; Partner systems
must use the relevant operational API rather than serialize them inside an
Offer.

The canonical conversion defaults are click window 720 hours, view window 0,
`last_click`, and `first` deduplication. Window endpoints are inclusive and zero
disables that source. Eligible clicks take precedence over eligible views.
`minimum_amount` is limited to integrations with one fixed reported conversion
currency; multi-currency thresholds remain internal policy.

`targeting[].language` accepts exactly two lowercase ASCII letters. This is a
stable syntax profile, not a claim that the value is an assigned ISO 639 code;
producers and consumers must not infer registry membership from Schema
acceptance.

The public and Partner carrier split in this document is normative.
The normative definition of every inherited and Partner-only property is in
[Offer and Query Response Field Semantics v1.0](offer-field-semantics.md);
the JSON Schema `description` annotations remain the authoritative field-level
contract.
