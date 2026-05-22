# Category Taxonomy v0.1

**Version**: 0.1
**Status**: Draft
**Last Updated**: 2026-04-23

## Purpose

This document is the **single source of truth** for the AgentOffer Protocol category surface.

It defines:

- the current **canonical public categories**
- accepted **input-only aliases**
- **future / later** categories when relevant
- **invalid / remove** values that must not appear in the public surface
- the current **public machine-readable contract** boundary

All human-readable and machine-readable category references in the v0.1 public surface should point here instead of redefining the category registry independently.

## Scope Boundary

This document does **not**:

- define IAB / DSP / SSP taxonomy mappings
- change the runtime `offer` or `query` field structure

Those follow-up responsibilities belong to later features:

- `Feature C`: external taxonomy mapping layer
- `Feature D`: downstream surface convergence

## Canonical Public Categories

The current public canonical category surface currently contains the following 11 values:

| Canonical value | Label | Description |
|-----------------|-------|-------------|
| `software_saas` | Software & SaaS | SaaS, subscription software, developer tools, AI tools |
| `travel_hospitality` | Travel & Hospitality | Hotels, flights, car rentals, vacation packages, dining experiences, attractions |
| `education` | Education & Learning | Courses, certifications, bootcamps, tutoring, academic programs |
| `financial_service` | Financial Services | Credit cards, insurance, loans, investment, banking, payments |
| `electronics` | Electronics & Devices | Consumer electronics, smart devices, wearables, gaming hardware |
| `entertainment` | Entertainment | Games, streaming, AI companions, audio, live entertainment |
| `health_beauty` | Health & Beauty | Skincare, supplements, cosmetics, wellness, fitness, medical-adjacent consumer offers |
| `fashion` | Fashion | Clothing, shoes, accessories, jewelry, luxury, sportswear |
| `food_grocery` | Food & Grocery | Meal kits, grocery delivery, specialty food, beverages, snacks |
| `home_garden` | Home & Garden | Furniture, appliances, decor, smart home, garden, cleaning |
| `automotive` | Automotive | Vehicle offers, leasing, insurance, parts, EV charging, ride services |

These 11 values are the only values that may currently appear in:

- public protocol docs as canonical categories
- public examples as canonical categories
- public machine-readable enums
- public prompt / action / integration guides as canonical output

## Accepted Aliases (Input-Only)

The following values may be accepted in parser / prompt / adapter normalization flows, but they are **not canonical public output**:

| Alias | Canonical target | Notes |
|-------|------------------|-------|
| `travel` | `travel_hospitality` | Historical shorthand |
| `hospitality` | `travel_hospitality` | Historical shorthand |
| `financial_services` | `financial_service` | Pluralized variant |
| `finance` | `financial_service` | Historical shorthand |
| `health_wellness` | `health_beauty` | Historical Action naming |
| `food_beverage` | `food_grocery` | Historical Action naming |
| `fashion_apparel` | `fashion` | Historical Action naming |
| `home_living` | `home_garden` | Historical Action naming |

Rules:

- aliases may be accepted in input
- aliases must normalize to canonical output
- aliases must not appear in public examples or public enums
- aliases must not be treated as a second public registry

## Future / Later Categories

The protocol may still introduce additional categories in later revisions.

Those values are not enumerated here until they are promoted into the canonical public surface.

## Invalid / Remove Values

The following values are not part of the current public category surface and should be removed from public exposure:

| Value | Reason | Allowed role |
|-------|--------|--------------|
| `real_estate` | Historical public exposure that does not belong to the current canonical 11 | None in public surface |
| `general` | Internal fallback bucket used by adapter logic; not a public taxonomy value | Internal fallback only |

## Value Classification Matrix

| Raw value | Current appearance | Classification | Normalized target | Allowed surface now |
|-----------|--------------------|----------------|-------------------|---------------------|
| `software_saas` | schema / docs / SDK / services | canonical | `software_saas` | input + output + docs + examples + public enum |
| `travel_hospitality` | schema / docs / SDK / services | canonical | `travel_hospitality` | input + output + docs + examples + public enum |
| `education` | schema / docs / SDK / services | canonical | `education` | input + output + docs + examples + public enum |
| `financial_service` | schema / docs / SDK / services | canonical | `financial_service` | input + output + docs + examples + public enum |
| `electronics` | schema / docs / SDK / services | canonical | `electronics` | input + output + docs + examples + public enum |
| `entertainment` | schema / docs | canonical | `entertainment` | input + output + docs + examples + public enum |
| `health_beauty` | protocol / schema / examples | canonical | `health_beauty` | input + output + docs + examples + public enum |
| `fashion` | protocol / schema / examples | canonical | `fashion` | input + output + docs + examples + public enum |
| `food_grocery` | protocol / schema / examples | canonical | `food_grocery` | input + output + docs + examples + public enum |
| `home_garden` | protocol / schema / examples | canonical | `home_garden` | input + output + docs + examples + public enum |
| `automotive` | protocol / schema / examples | canonical | `automotive` | input + output + docs + examples + public enum |
| `travel` | parser / prompt / Action historical copy | accepted alias | `travel_hospitality` | input-only |
| `hospitality` | parser / prompt | accepted alias | `travel_hospitality` | input-only |
| `financial_services` | parser / adapter | accepted alias | `financial_service` | input-only |
| `finance` | parser / adapter / historical copy | accepted alias | `financial_service` | input-only |
| `health_wellness` | historical Action enum | accepted alias | `health_beauty` | input-only |
| `food_beverage` | historical Action enum | accepted alias | `food_grocery` | input-only |
| `fashion_apparel` | historical Action enum | accepted alias | `fashion` | input-only |
| `home_living` | historical Action enum | accepted alias | `home_garden` | input-only |
| `real_estate` | historical Action enum | invalid / remove | none | remove from public surface |
| `general` | adapter fallback | invalid / internal fallback | none | internal fallback only |

## Public Machine-Readable Contract Rules

The current public machine-readable contract follows these rules:

1. Public enums currently allow only the 11 canonical category values.
2. Accepted aliases are input-only normalization helpers, not public enum members.
3. Invalid / remove values must not appear in public enums, public examples, or public canonical docs.
4. Internal fallbacks such as `general` must not be reinterpreted as public taxonomy categories.

## Relationship to Offer Schema

`offer_info.category.type` remains the field that carries the current AON product taxonomy.

This document defines:

- which values are canonical today
- which values are input-only aliases
- which values are future scope when relevant
- which values are invalid

The structural meaning of `offer_info.category` remains defined in [Offer Schema](./offer-schema.md).

## Relationship to External Taxonomies

IAB / DSP / SSP taxonomies are **not** the same thing as the AON category surface.

For v0.1:

- AON category taxonomy remains the internal canonical product taxonomy
- external taxonomies are treated as future adapter targets
- no external taxonomy mapping structure is defined in this document

## Downstream Consumers

This taxonomy document is the intended upstream reference for:

- SDK / Skill / Action category convergence
- service-side parser / adapter normalization rules
- AON-ORG public documentation alignment

Downstream follow-up actions are tracked in the monorepo planning and delivery docs, not in this public repository.
