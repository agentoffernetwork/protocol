# Offer Schema v0.3

> **Canonical v0.3 contract**

The machine-readable source is `offer-schema-v0.3.json`. Protocol v0.2 remains
the immutable compatibility contract and is intentionally unchanged.

## Public field groups

| Group | Meaning |
| --- | --- |
| `offer_info` | Display, category, commercial and descriptive content |
| `entity` | The merchant, brand, provider or publisher responsible for the offer |
| `listing_source` | The platform or site where the user-facing listing was observed |
| `action` | The destination and operation offered to the user |
| `claims` | Bounded advertiser or availability statements, not AON endorsement |
| `match_reason` | Optional concise explanation controlled by `thinking_mode` |

`listing_source` is supplemental source metadata, not a replacement for
`entity`, and is not inferred from the action URL. Internal provider identity,
affiliate parameters, eligibility decisions, freshness flags, mapping data and
commission data stay outside the public Offer.

v0.3 deliberately does not define `decision_factors`. The first release uses
`engagement.refinements` for narrowing the current intent and
`engagement.followup_topics` for adjacent exploration.
