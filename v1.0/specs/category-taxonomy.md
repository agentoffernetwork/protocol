# AON Taxonomy v1

**Version**: AON Taxonomy v1
**Status**: Stable shared current resource
**Last Updated**: 2026-09-10

## Purpose

This document is the single source of truth for the AgentOffer Protocol
category surface.

AON Taxonomy v1 defines stable dot-path `category.id` values for the current
public category surface.

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

- [Taxonomy tree](https://github.com/agentoffernetwork/schema/blob/main/v1.0/taxonomy/aon-taxonomy.json)
- [Taxonomy source schema](https://github.com/agentoffernetwork/schema/blob/main/v1.0/json-schema/taxonomy.schema.json)
- [Taxonomy resolver](https://github.com/agentoffernetwork/schema/blob/main/v1.0/taxonomy/aon-taxonomy-resolver.mjs)
- [Canonical metadata](https://github.com/agentoffernetwork/schema/blob/main/v1.0/taxonomy/aon-taxonomy-metadata.json)
- [Commerce candidate registry](https://github.com/agentoffernetwork/schema/blob/main/v1.0/taxonomy/commerce-product-candidates.json)
- [Warehouse commerce crosswalk](https://github.com/agentoffernetwork/schema/blob/main/v1.0/taxonomy/source-mappings/warehouse-commerce.json)
- [Commerce admission audit](https://github.com/agentoffernetwork/schema/blob/main/v1.0/taxonomy/audits/commerce-product-admission-audit.json)

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

The public tree is an admitted-only projection. Candidate presence in the
commerce registry or a source crosswalk does not make that id public or
selectable. Existing ids remain append-only, and a release adds a candidate
only after its own evidence and every candidate ancestor pass the admission
gate. A zero-admission evaluation produces an audit but no no-change taxonomy
release.

Canonical metadata owns the stable name, definition, aliases, examples,
subject boundary, and lifecycle of each node. Runtime status, sensitivity, and
display order remain operational state and are not inferred from the source
taxonomy or from warehouse lifecycle labels.

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

Depth is not a quality score. A broad parent remains selectable when it has
children and acts as the documented residual fallback when evidence cannot
support a narrower classification.

## Product, Platform, and Attribute Boundaries

A category identifies the primary subject and comparison model of an Offer. It
does not encode every property that could appear on a product page.

| Input concept | Taxonomy handling |
|---------------|-------------------|
| Stable sold product type | Use the most specific admitted product node supported by evidence |
| Marketplace or shopping platform itself | Use `e_commerce_marketplace`; do not classify listed products there |
| Brand, model, color, size, capacity, material, compatibility, ingredient, benefit, or style | Keep as product attributes; do not create combinatorial category ids |
| Delivery form such as physical product or online service | Use `offer_type` when applicable; it does not replace `category.id` |
| Service performed for a product | Use a service category when available; do not classify it as the underlying product |

Examples for mobile commerce:

| Offer subject | Category boundary |
|---------------|-------------------|
| Mobile phone handset | `internet_telecom.telephony.mobile_phones_accessories.mobile_phones` |
| Phone-only case, screen protector, replacement battery, or replacement part | A phone-specific admitted child under the mobile-phone branch |
| Generic charger, cable, power bank, or cross-device stand | An admitted node under `computers_electronics.consumer_electronics.consumer_electronic_accessories` |
| Tablet | A computer/electronics tablet node, never a mobile phone |
| Mobile subscription or phone plan | A telecom service node, never a physical phone product |

When evidence cannot distinguish phone-only use from cross-device use, the
Offer stays on the documented broad fallback rather than guessing a narrow id.

## Commerce Candidate Admission

Commerce expansion separates planning from publication:

1. The candidate registry freezes proposed ids, parents, provenance, semantic
   roles, and boundary fingerprints.
2. A source crosswalk gives every source category exactly one explicit action,
   including exact reuse, decomposition, broad fallback, platform mapping, and
   service exclusion.
3. The evaluator derives each candidate's risk tier, decision, reason, and
   evidence reference. The generated registry projection and audit must agree
   exactly.
4. Only a non-empty admitted set can produce a public tree delta and release
   manifest. Deferred candidates remain non-public.

All candidates require privacy-safe deduplicated positive samples, hard
negatives, a complete boundary review, and independent protected signoffs.
Warehouse-derived exact mappings may use one upstream namespace when all
inbound mappings are direct and exact. Split, broad-fallback, mixed-provenance,
and gap candidates require two independent namespaces plus decomposition and
residual evidence. Sensitive or regulated categories also require an explicit
operations sensitivity approval before activation.

Synthetic fixtures can test schema, projection, and failure behavior, but they
cannot satisfy production admission. Publishability is established only from
protected approval and immutable project, ref, job, and artifact provenance;
paths, filenames, output directories, caller flags, and test trust stores do
not confer publication authority.

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
- `category_types` is not a Taxonomy v1 public field.

Runtime subtree matching is evaluated against the release snapshot pinned at
the start of the request. The same snapshot governs primary and secondary
category validation, ranking, cached or remote candidates, the response, and
side effects. A runtime must not expand descendants from a newer embedded tree
while serving an older active release.

## Drift Guard

The source repository validates the current taxonomy before publication. A
consumer can also resolve and validate ids with the published
[taxonomy resolver](https://github.com/agentoffernetwork/schema/blob/main/v1.0/taxonomy/aon-taxonomy-resolver.mjs).

The source-tree guard is run by maintainers with:

```bash
node scripts/validate-taxonomy-v1.mjs
```

The guard:

1. Generates ids from `name + children`.
2. Checks id uniqueness.
3. Verifies required AON-owned ids.
4. Validates the checked-in resolver and mapping assets.
5. Scans examples for `offer_info.category.id`, `offer_info.secondary_category_ids`,
   and `category_ids`.
6. Fails on any id that does not exist in the registry.
7. Verifies candidate, metadata, crosswalk, audit, and release digests together.
8. Requires every warehouse source category to have exactly one crosswalk
   action and regenerates the human-readable comparison table from that source.
9. Confirms that a zero-admission run leaves the public tree and published
   consumer id sets unchanged.

Any taxonomy change must explicitly evaluate whether it adds, removes, moves,
renames, admits, defers, deprecates, or changes the boundary of a crosswalk
target. The crosswalk and generated comparison table change in the same commit
when required; otherwise the guard records a clean no-drift result.

## External Taxonomies

Google/GAM general categories were used as the initial baseline, but AON
Taxonomy v1 is AON-owned. Future changes to external taxonomies do not
automatically change the public AgentOffer Protocol.

IAB / DSP / SSP mappings remain future adapter work and are not part of this
Taxonomy v1 contract.
