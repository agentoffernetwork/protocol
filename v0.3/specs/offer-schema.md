# Offer Schema

> **Current stable contract**

The machine-readable source is the
[`offer-schema.json`](https://github.com/agentoffernetwork/schema/blob/main/v0.3/json-schema/offer-schema.json)
published with the current contract. Earlier compatibility material is retained
outside `main`.

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

When present, `listing_source.logo` is an optional platform/site logo. It must
be an absolute HTTPS URI no longer than 2048 characters; non-ASCII components
must be percent-encoded. It is never inferred
from `entity.logo`, `action.payload.url`, or `material[]`; when absent,
consumers use `listing_source.name` as the display and accessibility fallback.
`listing_source.url` remains outside the closed public object.

The current contract deliberately does not define `decision_factors`. It uses
`engagement.refinements` for narrowing the current intent and
`engagement.followup_topics` for adjacent exploration.
