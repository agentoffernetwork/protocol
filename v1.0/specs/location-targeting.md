# Location and Age Targeting

This shared specification defines the current Offer-side shape for geographic
targeting and minimum-age eligibility. It does not define a public viewer
location or age input for Query v1.0.

## Offer shape

An Offer may publish up to ten `targeting[]` rules. A rule may contain
`geo.include`, `geo.exclude`, and `eligibility.min_age` alongside the current
language, device, and operating-system fields.

```json
{
  "targeting": [
    {
      "geo": {
        "include": [{"location_id": "2840"}],
        "exclude": [{"location_id": "21180"}]
      },
      "eligibility": {"min_age": 18}
    }
  ]
}
```

Each location entry is a closed object containing exactly one numeric-string
`location_id`. Use ids from the current
[AON Location Registry](https://github.com/agentoffernetwork/schema/blob/main/v1.0/locations/aon-location-registry.json).
Display names, country strings, coordinates, and external subdivision codes
are not substitutes for `location_id` in an Offer.

`eligibility.min_age` is an integer from 13 through 120. It declares an Offer
eligibility threshold; it is not a request to collect or infer a person's date
of birth.

The exact structural source is the current
[Offer JSON Schema](https://github.com/agentoffernetwork/schema/blob/main/v1.0/json-schema/offer-schema.json).

## Query boundary

The current Query v1.0 request may carry an optional viewer location and a
verified minimum-age assertion in `context.user_profile`:

```json
{
  "context": {
    "user_profile": {
      "location_ids": ["1014221"],
      "country": "US",
      "verified_age_over": 21
    }
  }
}
```

- `location_ids`: one to ten unique numeric-string `location_id` values from
  the AON Location Registry, most specific first. The runtime expands each
  resolvable id to its ancestor chain before matching `geo.include` and
  `geo.exclude`.
- `country`: an uppercase ISO 3166-1 alpha-2 code, kept for callers that only
  know the viewer country. It is used when `location_ids` is absent or does not
  resolve; a resolvable `location_ids` entry supersedes it.

Both location fields are optional. When neither is supplied, the runtime may
fall back to deployment-specific signals such as an edge-provided country
header.

- `verified_age_over`: an integer from 13 through 120 asserting that the
  integrator has verified the viewer is at least that age. It is an age
  threshold, never the actual age or date of birth, and must come from a real
  verification process such as account identity checks or a platform age
  rating, not from inference.

Age is evaluated only when the request supplies it. A rule with
`eligibility.min_age` fails when the request carries a `verified_age_over`
below `min_age`. When `verified_age_over` is omitted, age is not evaluated and
`min_age` does not restrict the request. Supplying the field can therefore only
narrow results; it never unlocks an Offer an unverified request would not
receive.

The Query request still does **not** expose coordinates, display names, an
exact age, or a date of birth. Clients must not infer location or age from
free text, add undeclared viewer fields, or treat Offer targeting declarations
as caller-supplied Query filters. Earlier location-search material
is retained only on immutable historical refs and is not part of the current
v1.0 tree.
