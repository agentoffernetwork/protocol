# AON Taxonomy v1

**Version**: AON Taxonomy v1
**Status**: Stable shared current resource
**Last Updated**: 2026-08-27

## Purpose

This document is the single source of truth for the AgentOffer Protocol
category surface.

AON Taxonomy v1 replaces the old v0.1 `category.type` +
`category.attributes.sub_type` model with stable dot-path `category.id` values.

Public Offer payloads reference a selected taxonomy node:

```json
{
  "offer_info": {
    "category": {
      "id": "arts_entertainment.igaming"
    }
  }
}
```

## Machine-Readable Source

The current machine-readable sources live in the schema repository:

- [Taxonomy tree](https://github.com/agentoffernetwork/schema/blob/main/v0.3/taxonomy/aon-taxonomy.json)
- [Taxonomy source schema](https://github.com/agentoffernetwork/schema/blob/main/v0.3/json-schema/taxonomy.schema.json)
- [Taxonomy resolver](https://github.com/agentoffernetwork/schema/blob/main/v0.3/taxonomy/aon-taxonomy-resolver.mjs)

Source nodes use only:

| Field | Description |
|-------|-------------|
| `name` | Human-facing category name |
| `children` | Child category nodes |

Stable ids are generated from the node path by the taxonomy guard:

```text
Arts & Entertainment > iGaming
=> arts_entertainment.igaming
```

The generated id is the only category value Partner-written Offer payloads need
to carry.

## Level 1 Canonical IDs

| Source Level 1 | AON display name | AON canonical id |
|----------------|------------------|------------------|
| Apparel | Fashion & Apparel | `fashion_apparel` |
| Arts & Entertainment | Arts & Entertainment | `arts_entertainment` |
| Autos & Vehicles | Automotive | `automotive` |
| Beauty & Personal Care | Beauty & Personal Care | `beauty_personal_care` |
| Business & Industrial | Business & Industrial | `business_industrial` |
| Computers & Consumer Electronics | Computers & Electronics | `computers_electronics` |
| Dining & Nightlife | Dining & Nightlife | `dining_nightlife` |
| E-commerce & Marketplace | E-commerce & Marketplace | `e_commerce_marketplace` |
| Family & Community | Family & Community | `family_community` |
| Finance | Finance | `finance` |
| Food & Groceries | Food & Grocery | `food_grocery` |
| Health | Health | `health` |
| Hobbies, Games & Leisure | Hobbies, Games & Leisure | `hobbies_games_leisure` |
| Home & Garden | Home & Garden | `home_garden` |
| Internet & Telecom | Internet & Telecom | `internet_telecom` |
| Jobs & Education | Jobs & Education | `jobs_education` |
| Law & Government | Law & Government | `law_government` |
| Mobile App Utilities | Mobile Utilities | `mobile_utilities` |
| News, Books & Publications | News, Books & Publications | `news_books_publications` |
| Occasions & Gifts | Gifts & Occasions | `gifts_occasions` |
| Others | Others | `others` |
| Real Estate | Real Estate | `real_estate` |
| Sports & Fitness | Sports & Fitness | `sports_fitness` |
| Travel & Tourism | Travel & Tourism | `travel_tourism` |

Slug rules:

1. Remove low-value connector words such as `and` / `&`.
2. Preserve words that carry business meaning.
3. Prefer short, stable, readable ids over mechanically generated strings.
4. Review Level 1 and high-volume Level 2 nodes manually before release.

## Category Depth

| Level | Meaning | Example |
|-------|---------|---------|
| Level 1 | Top-level business category | `travel_tourism` |
| Level 2 | Optional narrower category | `finance.credit_lending` |
| Level 3+ | Optional detailed category | `computers_electronics.computers.software` |

Partner entry rule:

- Level 1 is required.
- Level 2 and Level 3+ are optional.
- Search, adapter mapping, system suggestion, or Admin review may fill deeper
  category ids.

## E-commerce & Marketplace Disambiguation

The `e_commerce_marketplace` chain categorizes offers that describe the
**marketplace / platform entity itself** — the shopping channel — not the
individual products listed on it.

| Offer subject | AON canonical id |
|---------------|------------------|
| Comprehensive e-commerce platform (e.g. Amazon, JD) | `e_commerce_marketplace.comprehensive_e_commerce_platform` |
| B2C marketplace (e.g. Temu, Shein) | `e_commerce_marketplace.comprehensive_e_commerce_platform.b2c_marketplace` |
| Generic / unspecified e-commerce marketplace | `e_commerce_marketplace` |

Specific **products** sold on these platforms must still be categorized under
their own product vertical, including that vertical's existing online-shopping
child (for example `Food & Groceries > Online Grocery Shopping`), not under
`e_commerce_marketplace`. This keeps the platform/channel axis separate from the
product-vertical axis and prevents dual-classification ambiguity.

## Short Drama Scope

Short Drama (`arts_entertainment.short_drama`) covers vertical / mobile-first
serialized micro-dramas, distinct from long-form Movies & Films and broadcast
TV & Video.

## Sensitive Category Boundary

Sensitive category handling is split from public Offer payload shape.

| Semantics | Public contract |
|-----------|-----------------|
| What the Offer is | `offer_info.category.id` |
| Geo / language / device / OS applicability | Existing top-level `targeting` |
| Sensitive review requirement | Internal platform rule derived from `category.id` |
| Public policy flags | Not part of the first Taxonomy v1 Offer payload |

Required AON-owned public taxonomy ids:

| Category id | Notes |
|-------------|-------|
| `arts_entertainment.adult_entertainment` | Public adult category; platform derives internal sensitive review |
| `arts_entertainment.igaming` | Real-money iGaming / online casino / betting category |
| `finance.investing.crypto_and_digital_assets` | Crypto trading, digital asset investment, and related services under Finance > Investing |

## Query and Provider Constraints

Query API and OfferProvider API category constraints use `category_ids`:

```json
{
  "constraints": {
    "category_ids": ["arts_entertainment.igaming"]
  }
}
```

Matching semantics:

- OR logic within the array.
- Each id matches the selected taxonomy node and its descendants.
- Matching checks an offer's primary `offer_info.category.id` and any optional
  `offer_info.secondary_category_ids` using the same subtree semantics.
- Secondary ids are auxiliary cross-branch classifications. They must not repeat
  the primary category and must not be an ancestor or descendant of the primary
  category or another secondary id.
- `others` is a standard Level 1 id with no child categories in this version,
  so `category_ids=["others"]` currently matches only stored id `others`.
- Category ids are case-sensitive; use lowercase canonical ids such as `others`.
- `category_types` is legacy migration language only and is not the Taxonomy v1
  public path.

## v0.1 Migration

The old 11-value v0.1 category model is retained only for migration:

- `software_saas`
- `travel_hospitality`
- `education`
- `financial_service`
- `electronics`
- `entertainment`
- `health_beauty`
- `fashion`
- `food_grocery`
- `home_garden`
- `automotive`

The historical migration assets remain available from the verified
[v0.2 legacy tag](https://github.com/agentoffernetwork/schema/tree/v0.2.0-legacy).

When a legacy `category.type + attributes.sub_type` pair cannot map precisely,
it should map to the closest stable parent node instead of inventing a new
public id.

## Drift Guard

The source repository validates the current taxonomy before publication. A
consumer can also resolve and validate ids with the published
[taxonomy resolver](https://github.com/agentoffernetwork/schema/blob/main/v0.3/taxonomy/aon-taxonomy-resolver.mjs).

The source-tree guard is run by maintainers with:

```bash
node scripts/validate-taxonomy-v1.mjs
```

The guard:

1. Generates ids from `name + children`.
2. Checks id uniqueness.
3. Verifies required AON-owned ids.
4. Validates v0.1 migration targets.
5. Scans examples for `offer_info.category.id`, `offer_info.secondary_category_ids`,
   and `category_ids`.
6. Fails on any id that does not exist in the registry.

## External Taxonomies

Google/GAM general categories were used as the initial baseline, but AON
Taxonomy v1 is AON-owned. Future changes to external taxonomies do not
automatically change the public AgentOffer Protocol.

IAB / DSP / SSP mappings remain future adapter work and are not part of this
Taxonomy v1 contract.
