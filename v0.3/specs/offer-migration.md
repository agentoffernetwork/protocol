# Final v0.3 Producer Migration and Rejection Guide

## Scope and effective point

This guide applies to the immutable final v0.3 candidate selected by protected
release preparation. It does not rewrite `protocol-v0.3.0-r12` or any historical
release evidence. Producers must make payloads conform before advertising that
candidate.

## Required migration

| Previous accepted shape | Final v0.3 requirement | Rejection behavior |
| --- | --- | --- |
| Invalid language label such as `english`, `en_US`, or a tag with a repeated extension singleton | Use the final-v0.3 BCP-47 profile, such as `en` or `en-US-u-hc-h12` | Reject with `language_bcp47` |
| Duplicate `goals[].event` | Keep one Goal per event | Reject with `event_unique` |
| CPA `amount: "0"` | Send a strictly positive canonical decimal | Reject with `amount_positive` |
| CPS `rate: "0"` | Send a strictly positive percentage no greater than 100 | Reject with `rate_positive` |
| `start_at > expire_at` | Send an ordered availability window | Reject with `offer_window_order` |
| HTTP or relative logos | Send an absolute HTTPS logo URI or omit the logo | Reject with `logo_https` |
| HTTP, relative, credential-bearing, or executable resource URI | Use an absolute HTTPS resource URI without userinfo or omit the optional resource | Reject with `resource_https` |
| Unsafe `action.payload.url` or non-HTTPS `open_url` | Use the action-type safe URI profile | Reject with `action_uri_safe` |
| Price labels such as `free!` | Send a canonical decimal string; use `"0"` only where a zero public price is intended | Reject with `price_decimal` |
| Empty Partner targeting/configuration | Omit unused fields or provide non-empty rules | Reject with `targeting_nonempty` or `conversion_rule_nonempty` |
| Numeric but unregistered Partner `location_id` | Resolve and send an id present in the pinned AON Location Registry v1 snapshot | Reject with `location_registry_membership` |
| Partner `offer_id`, `offer_instance_id`, or `match_reason` | Send stable `source_offer_id` and remove AON-authored public-response fields; AON resolves/adds them during projection | Reject with `aon_projection_field` or the closed Partner Schema |
| Query Helper `null` removal | Omit unchanged fields or send non-null replacement values | Reject the Query Helper patch |
| Query Helper empty nested object | Include an actual signal or category-constraint replacement; an empty replacement array may intentionally clear one category list | Reject the Query Helper patch |
| Unknown `display_pattern` token | Use only `${type}`, `${value}`, and `${unit}` | Reject with `display_pattern_token` |
| Empty optional descriptive string or `rating.count: 0` | Omit unavailable copy/count; when supplied, descriptive copy is non-empty and rating count is positive | Reject under the public Offer Schema |
| Public `targeting` / `conversion_rule` | Send them only in the Partner Offer artifact | Public carrier rejects the closed-schema payload |
| `status`, `audit_status`, `priority` inside an Offer | Use operator policy/operational APIs | Public and Partner carriers reject the closed-schema payload |

## Validation order

1. Validate the selected public or Partner JSON Schema.
2. Run `validatePublicOfferV03Semantics` or
   `validatePartnerOfferV03Semantics` with the pinned AON Taxonomy v1 and AON
   Location Registry v1 snapshots.
3. Reject the payload if either layer fails. Do not sanitize an invalid payload
   and claim it was canonical conformance.

These structured error codes are canonical validation results, not hosted HTTP
status codes. Runtime teams may map them to endpoint errors after they adopt the
contract, but such implementation work does not change the protocol result.
