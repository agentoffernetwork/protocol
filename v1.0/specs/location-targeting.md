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

The current Query v1.0 request exposes bounded platform and session context,
structured intent, category constraints, and response controls. It does **not**
expose viewer `location_id`, `location_ids`, country, coordinates,
`verified_age_over`, or another viewer-age field.

Clients must not add undeclared viewer location or age fields to the Query
body, infer them from free text, or treat Offer targeting declarations as
caller-supplied Query filters. Deployment-specific eligibility inputs and
runtime policy remain outside this portable public request contract. Earlier
location-search material is retained only on immutable historical refs and is
not part of the current v1.0 tree.
