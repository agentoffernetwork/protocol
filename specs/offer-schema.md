# Offer Schema v0.1

**Version**: 0.1
**Status**: Draft
**Last Updated**: 2026-03-22

## Introduction

This document defines the canonical `Offer` object for AgentOffer Protocol v0.1. It is aligned with the canonical machine-readable schema and type definitions maintained for AgentOffer, and is intended to be the normative human-readable companion to the JSON Schema and TypeScript model.

The schema is optimized for two goals:

- settlement accuracy, including commission calculation, attribution, and reconciliation
- matching quality, including semantic relevance, ranking signals, and negative filtering for agent recommendations

It is also intended to stay adaptable to existing industry formats such as Schema.org commerce markup, Google Merchant-style feeds, GS1 product identifiers, and OpenRTB-style extension patterns.

## Required Top-Level Fields

| Field | Type | Description |
|------|------|-------------|
| `id` | string | Unique offer identifier. Format: `ao_{ulid}`. |
| `title` | string | Product or service name suitable for agent display. |
| `description` | string | Long-form product or service description used for semantic matching. |
| `url` | string | Direct advertiser landing page URL, not a tracking link. |
| `price` | object | Current selling price with `amount` and `currency`. |
| `currency` | string | Convenience currency field matching `price.currency`. |
| `commission` | object | Commission and settlement definition. |
| `advertiser` | object | Advertiser identity and settlement reference. |
| `category` | object | Primary product taxonomy. |
| `status` | enum | Current lifecycle status in AgentOffer. |

## Object Model

### Identity

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `id` | string | Yes | Unique offer ID with pattern `^ao_[0-9A-Za-z]{26}$`. |
| `sku` | string | No | Merchant-facing stock keeping unit or seller-specific product identifier. |
| `variant_group_id` | string | No | Stable identifier used to group product variants. This is the closest AgentOffer equivalent to Google Merchant `item_group_id` and Schema.org `inProductGroupWithID` / `productGroupID`. |
| `external_ids` | object | No | External platform identifiers for deduplication and reconciliation. |

Supported `external_ids` keys include:

- `impact_catalog_item_id`
- `cj_ad_id`
- `shareasale_sku`
- `partnerstack_deal_key`
- `advertiser_sku`
- `gtin`
- `upc`
- `isbn`
- `asin`
- `mpn`

Notes:

- When available, `gtin` should contain a GS1-compatible GTIN and must not be guessed.
- `mpn` should contain the manufacturer-assigned part number when applicable.
- `sku` is distinct from `external_ids.advertiser_sku`; `sku` is the canonical merchant-facing identifier in the AgentOffer profile.

### Core Content

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `title` | string | Yes | Product or service name, max length 200. |
| `description` | string | Yes | Full product description, max length 5000. |
| `short_description` | string | No | One-sentence summary optimized for conversational recommendation, max length 300. |
| `url` | string | Yes | Advertiser landing page URL. |

### Pricing

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `price.amount` | number | Yes | Current price amount. Use `0` for free products or trials. |
| `price.currency` | string | Yes | ISO 4217 currency code such as `USD`, `EUR`, or `GBP`. |
| `currency` | string | Yes | Same logical currency as `price.currency`, exposed as a convenience field. |
| `original_price` | number | No | Retail price before discount. |
| `discount_percentage` | number | No | Discount percent from `original_price` to `price`. |
| `pricing_model` | enum | No | Pricing model for the product or service. |
| `free_trial_days` | integer | No | Number of trial days, if applicable. |

`pricing_model` values:

- `one_time`
- `subscription_monthly`
- `subscription_yearly`
- `freemium`
- `free_trial`
- `pay_per_use`
- `custom`

### Commission and Settlement

The commission model is an object, not a pair of flat fields. This is the biggest difference from the earlier simplified draft.

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `commission.type` | enum | Yes | Calculation method: `percentage`, `fixed`, `tiered`, or `hybrid`. |
| `commission.value` | number | Yes | Decimal percentage or fixed payout amount, depending on `type`. |
| `commission.max_amount` | number | No | Maximum payout cap per conversion. |
| `commission.payout_level` | enum | No | Payout trigger level. |
| `commission.cookie_window_days` | integer | No | Attribution window in days. |
| `commission.recurring` | boolean | No | Whether payout recurs on subscription renewals. |
| `commission.recurring_months` | integer | No | Number of recurring months, if limited. |
| `commission.validation_window_days` | integer | No | Days before a conversion is confirmed for settlement. |
| `commission.conversion_criteria` | object | No | Rules that define a valid conversion. |
| `commission.tiers` | array | No | Tier rules used only when `type = tiered`. |
| `commission.special_terms` | string | No | Human-readable payout constraints or notes. |

`commission.payout_level` values:

- `item`
- `order`
- `click`
- `lead`
- `install`

`commission.conversion_criteria` may include:

- `min_order_amount`
- `new_customer_only`
- `excluded_skus`
- `excluded_categories`
- `required_actions`

Each `commission.tiers[]` object supports:

- `min_conversions`
- `max_conversions`
- `value`
- `value_type` with values `percentage` or `fixed`

### Advertiser

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `advertiser.id` | string | Yes | Advertiser ID inside AgentOffer. |
| `advertiser.name` | string | Yes | Advertiser display name. |
| `advertiser.domain` | string | No | Primary advertiser hostname. |
| `advertiser.logo_url` | string | No | Advertiser logo URL. |
| `advertiser.verified` | boolean | No | Whether the advertiser identity is verified. |
| `advertiser.settlement_terms_id` | string | No | Reference to advertiser-level settlement terms configuration. |

### Source and Reconciliation

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `source.type` | enum | No | Source platform or acquisition path. |
| `source.platform_offer_id` | string | No | Upstream program or offer ID. |
| `source.platform_advertiser_id` | string | No | Upstream advertiser ID. |
| `source.program_name` | string | No | Affiliate program name on source platform. |
| `source.postback_url_template` | string | No | Conversion callback template used for settlement and reconciliation. |
| `source.reconciliation_frequency` | enum | No | Sync cadence with upstream source. |
| `source.tracking_url_template` | string | No | Upstream tracking URL template. |
| `source.imported_at` | string | No | Import timestamp. |
| `source.last_synced_at` | string | No | Last successful sync timestamp. |

`source.type` values:

- `direct`
- `impact`
- `cj`
- `shareasale`
- `partnerstack`
- `awin`
- `other_affiliate`

`source.reconciliation_frequency` values:

- `realtime`
- `hourly`
- `daily`
- `weekly`

### Category and Taxonomy

`category` is structured, not a plain string.

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `category.primary` | enum | Yes | AgentOffer standard taxonomy category. |
| `category.secondary` | string | No | More specific category. |
| `category.advertiser_category` | string | No | Advertiser-native category. |
| `category.breadcrumb` | string | No | Hierarchical category path from source platform. |

`category.primary` values:

- `saas_tools`
- `developer_tools`
- `design_tools`
- `productivity`
- `marketing`
- `finance`
- `education`
- `health_wellness`
- `electronics`
- `fashion`
- `home_garden`
- `food_beverage`
- `travel`
- `entertainment`
- `gaming`
- `other`

### Tags

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `tags` | array of string | No | Lowercase, hyphenated tags for filtering and semantic matching. Max 20 unique items. |

### Agent Recommendation Metadata

This section is essential for recommendation quality and was entirely missing from the simplified draft.

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `agent_metadata.recommendation_context` | string | No | Primary semantic matching field describing when the offer should be recommended. |
| `agent_metadata.use_cases` | array | No | Structured recall use cases. |
| `agent_metadata.key_features` | array | No | Key features agents may highlight in conversation. |
| `agent_metadata.comparison_points` | object | No | Structured comparison data for "X vs Y" style questions. |
| `agent_metadata.target_audience` | string | No | Who this offer is meant for. |
| `agent_metadata.not_suitable_for` | string | No | Who should not receive this recommendation. |
| `agent_metadata.alternatives` | array | No | Known alternatives. |
| `agent_metadata.talking_points` | array | No | Pre-written recommendation phrasing. |
| `agent_metadata.trigger_keywords` | array | No | Rule-based positive recall terms. |
| `agent_metadata.negative_keywords` | array | No | Rule-based suppression terms. |
| `agent_metadata.suitable_agent_types` | array | No | Agent classes that this offer fits. |

`agent_metadata.suitable_agent_types` values:

- `productivity-assistant`
- `tech-advisor`
- `shopping-assistant`
- `travel-planner`
- `finance-advisor`
- `health-coach`
- `education-tutor`
- `creative-assistant`
- `developer-tools`
- `general-purpose`

### Media

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `media.image_url` | string | No | Primary product image. |
| `media.thumbnail_url` | string | No | Thumbnail image. |
| `media.additional_images` | array | No | Additional image URLs. |
| `media.video_url` | string | No | Product video URL. |

### Status and Lifecycle

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `status` | enum | Yes | Offer lifecycle status. |
| `availability` | enum | No | Availability state. |
| `geo_restrictions.allowed_countries` | array | No | ISO 3166-1 alpha-2 countries where the offer is available. |
| `geo_restrictions.blocked_countries` | array | No | ISO 3166-1 alpha-2 countries where the offer is unavailable. |
| `start_date` | string | No | Activation timestamp. |
| `end_date` | string | No | Expiration timestamp. |

Compatibility notes:

- `start_date` is the closest AgentOffer equivalent to Schema.org `validFrom`.
- `end_date` is the closest AgentOffer equivalent to Schema.org `validThrough` and merchant validity windows.

`status` values:

- `draft`
- `pending_review`
- `active`
- `paused`
- `expired`
- `rejected`

`availability` values:

- `in_stock`
- `out_of_stock`
- `preorder`
- `limited`
- `always_available`

### Compliance

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `compliance.disclosure_required` | boolean | No | Whether agents must disclose a commercial relationship. |
| `compliance.disclosure_text` | string | No | Recommended disclosure text for this offer. |
| `compliance.terms_url` | string | No | Terms and conditions URL. |
| `compliance.restricted_claims` | array | No | Claims agents must not make. |

### Quality and Performance

These are platform-computed, mostly read-only signals used for ranking and trust.

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `quality.platform_rating` | number | No | Internal platform quality score from 0 to 5. |
| `quality.external_rating` | number | No | External review score from 0 to 5. |
| `quality.external_review_count` | integer | No | External review count. |
| `quality.refund_rate` | number | No | Historical refund rate as decimal. |
| `quality.conversion_rate` | number | No | All-time click-to-conversion rate. |
| `quality.conversion_rate_7d` | number | No | Rolling 7-day conversion rate. |
| `quality.conversion_rate_30d` | number | No | Rolling 30-day conversion rate. |
| `quality.conversion_trend` | enum | No | `rising`, `stable`, or `declining`. |
| `quality.click_count_30d` | integer | No | Click volume over the last 30 days. |
| `quality.earnings_per_click` | number | No | Average earnings per click. |
| `quality.featured` | boolean | No | Whether the offer is manually featured. |

### Fulfillment

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `fulfillment.type` | enum | No | Delivery type. |
| `fulfillment.shipping_cost` | number | No | Standard shipping cost. |
| `fulfillment.free_shipping_threshold` | number | No | Minimum basket value for free shipping. |
| `fulfillment.estimated_delivery_days` | integer | No | Delivery lead time in business days. |
| `fulfillment.return_policy_days` | integer | No | Return window in days. |

`fulfillment.type` values:

- `digital`
- `physical`
- `service`
- `subscription`

Compatibility notes:

- This section is intentionally lighter than Schema.org `OfferShippingDetails` and Google shipping-policy markup.
- Implementations that need rich shipping or return-policy export should use `fulfillment` for core ranking signals and place richer adapter-specific details under `ext.schema_org` or `ext.google_merchant`.

### Physical Product Attributes

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `attributes.brand` | string | No | Brand name. |
| `attributes.manufacturer` | string | No | Manufacturer name. |
| `attributes.color` | string | No | Product color. |
| `attributes.size` | string | No | Product size label. |
| `attributes.material` | string | No | Material information. |
| `attributes.condition` | enum | No | Condition state. |
| `attributes.weight.value` | number | No | Product weight value. |
| `attributes.weight.unit` | enum | No | Weight unit. |

`attributes.condition` values:

- `new`
- `used`
- `refurbished`
- `open_box`

`attributes.weight.unit` values:

- `g`
- `kg`
- `oz`
- `lb`

### Extensibility and Metadata

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `ext` | object | No | Extension object for non-standard fields using OpenRTB-style extensibility. |
| `schema_version` | string | No | Schema version constant. In v0.1 this must be `0.1`. |
| `created_at` | string | No | Offer creation timestamp in AgentOffer. |
| `updated_at` | string | No | Offer last update timestamp. |

Recommended extension namespaces:

- `ext.schema_org` for export-oriented markup details such as `businessFunction`, `shippingDetails`, or richer return-policy objects
- `ext.google_merchant` for feed-specific attributes that should not become core protocol fields
- `ext.openrtb_native` for native-ad or decomposed asset adapter payloads

## Industry Compatibility

### Schema.org / Merchant Listing Alignment

The current AgentOffer model aligns most naturally with `Product` + `Offer` style markup:

| AgentOffer | Closest external concept |
|------------|--------------------------|
| `title` | `Product.name` |
| `description` | `Product.description` |
| `url` | canonical product URL |
| `sku` | `Offer.sku` |
| `price.amount` | `Offer.price` |
| `price.currency` / `currency` | `Offer.priceCurrency` |
| `availability` | `Offer.availability` |
| `attributes.brand` | `Product.brand` |
| `attributes.condition` | `itemCondition` |
| `start_date` / `end_date` | `validFrom` / `validThrough` |
| `variant_group_id` | `inProductGroupWithID` / `productGroupID` / Google `item_group_id` |

### Google Merchant Feed Alignment

The following fields are especially important when adapting AgentOffer offers into merchant-style feeds:

- `sku`
- `variant_group_id`
- `attributes.brand`
- `external_ids.gtin`
- `external_ids.mpn`
- `attributes.condition`
- `availability`
- `geo_restrictions.allowed_countries` / `blocked_countries`

### GS1 Identifier Guidance

- If `external_ids.gtin` is present, it should be treated as a high-trust global identifier.
- Implementations should validate GTIN structure and, where operationally feasible, verify against GS1 / Verified by GS1 workflows.
- `gtin` should not be generated from guesses, retailer scraping, or weak heuristics.

### OpenRTB Native / Ad-Tech Alignment

AgentOffer is not an ad bid schema, but it should remain adaptable to decomposed native inventory objects:

- `title` maps naturally to title assets
- `description` and `short_description` map naturally to body/text assets
- `media` maps naturally to image assets
- `compliance.disclosure_*` can inform ad-disclosure or sponsored-label rendering
- `ext` should remain the main compatibility surface for OpenRTB-specific export details

## Validation Rules

- `id` must match `^ao_[0-9A-Za-z]{26}$`.
- `sku`, when present, should be stable within the seller or source system and should not be reused across unrelated products.
- `variant_group_id`, when present, should be stable across all variants in the same family.
- `url`, `logo_url`, media URLs, and terms URLs must be valid URIs when present.
- `price.currency` and `currency` must use uppercase ISO 4217 codes.
- `commission.value` uses decimal notation for percentage commissions, for example `0.20` means 20%.
- `tags` must be lowercase and hyphenated, with no duplicates.
- `geo_restrictions.allowed_countries` and `blocked_countries` must use ISO 3166-1 alpha-2 country codes.
- `external_ids.gtin`, when present, should use a normalized GTIN value without display formatting.
- `schema_version`, when present, must equal `0.1`.
- The reference JSON Schema sets `additionalProperties: false` at the top level; implementations should not invent top-level fields outside this model.

## Example

```json
{
  "id": "ao_01HX2B3C4D5E6F7G8H9J0KABCD",
  "sku": "notion-team-monthly-usd",
  "variant_group_id": "notion-team-plan",
  "title": "Notion Team Plan",
  "description": "Notion is an all-in-one workspace that combines notes, documents, knowledge bases, project management, and collaboration tools.",
  "short_description": "All-in-one workspace for notes, docs, and project management — the operating system for your team.",
  "url": "https://www.notion.so/product",
  "price": {
    "amount": 10,
    "currency": "USD"
  },
  "currency": "USD",
  "pricing_model": "subscription_monthly",
  "free_trial_days": 0,
  "commission": {
    "type": "percentage",
    "value": 0.3,
    "payout_level": "order",
    "cookie_window_days": 30,
    "recurring": true,
    "recurring_months": 12,
    "validation_window_days": 30,
    "conversion_criteria": {
      "min_order_amount": 0,
      "new_customer_only": true,
      "required_actions": [
        "purchase",
        "no_refund_14d"
      ]
    }
  },
  "advertiser": {
    "id": "adv_notion_001",
    "name": "Notion Labs, Inc.",
    "domain": "notion.so",
    "verified": true,
    "settlement_terms_id": "st_default_net30"
  },
  "source": {
    "type": "impact",
    "platform_offer_id": "12345",
    "program_name": "Notion Affiliate Program"
  },
  "category": {
    "primary": "saas_tools",
    "secondary": "project_management"
  },
  "tags": [
    "project-management",
    "team-collaboration",
    "note-taking"
  ],
  "agent_metadata": {
    "recommendation_context": "Recommend when the user needs a team collaboration tool or all-in-one workspace.",
    "target_audience": "Small to mid-size teams looking for a shared workspace.",
    "negative_keywords": [
      "self-hosted",
      "offline only"
    ]
  },
  "media": {
    "image_url": "https://www.notion.so/images/meta/default.png"
  },
  "status": "active",
  "availability": "always_available",
  "compliance": {
    "disclosure_required": true,
    "disclosure_text": "Sponsored recommendation"
  },
  "schema_version": "0.1",
  "created_at": "2026-03-15T00:00:00Z",
  "updated_at": "2026-03-16T12:00:00Z"
}
```

## Design Decisions

- The schema uses nested objects such as `commission`, `advertiser`, `source`, and `agent_metadata` because the protocol must support both marketplace settlement workflows and high-quality recommendation matching.
- `url` is the canonical destination page, while source-specific tracking templates live under `source`; this keeps content identity separate from attribution mechanics.
- `category` is structured so the platform can preserve both AgentOffer taxonomy and advertiser-native classifications.
- Recommendation-specific metadata is part of the offer model because matching quality depends on richer context than catalog metadata alone.
- Platform-computed `quality` fields are included so ranking and trust signals can be distributed without redefining a second analytics object.
- `sku` and `variant_group_id` are included as optional interoperability fields because merchant feeds and structured-data adapters depend on stable seller identifiers and variant family grouping.
- Rich shipping, return, and external markup details are intentionally kept out of the core schema and routed through `ext` adapter namespaces unless repeated implementation demand justifies promotion into the base model.

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-03-20 | Initial draft. |
| 0.1 | 2026-03-20 | Rewritten to align with the canonical machine-readable schema and type definitions, replacing the earlier simplified field model. |
| 0.1 | 2026-03-22 | Added interoperability guidance for Schema.org, Google Merchant, GS1, and OpenRTB-style adapters; added optional `sku` and `variant_group_id` fields. |
