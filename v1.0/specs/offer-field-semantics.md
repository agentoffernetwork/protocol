# Offer and Query Response Field Semantics v1.0

> **Final canonical field-level reference**

This reference defines the meaning, authority, and cross-field behavior of every
public Offer, normalized Partner Offer, and Query response property in final
v1.0. The `description` annotation on each JSON Schema `properties` entry is
normative. This document supplies the shared rules and value glossaries that do
not fit in one field annotation.

## Authority and reading rules

| Layer | Authority | Use |
| --- | --- | --- |
| Public Offer fields | `offer-schema-v1.0.json` property descriptions | Agent-facing Offer content, dispatch identity, and public Goal commission declarations |
| Partner Offer fields | `offer-partner-schema-v1.0.json` property descriptions | Normalized Partner supply content, targeting, and attribution configuration |
| Query response fields | `offer-query-response-v1.0.json` property descriptions | Response envelope, ordering, guidance, and no-result semantics |
| Cross-field constraints | v1.0 semantic validator and this specification | Rules that JSON shape alone cannot express |

An omitted optional field means that the producer makes no declaration for that
field. Stable v1.0 does not use `null` as a substitute for omission. Empty arrays
and empty objects are invalid when they would carry no information. Consumers
must not infer omitted provenance, targeting, pricing, eligibility, freshness,
or private ranking data.

Optional descriptive strings defined with `minLength: 1` must be omitted when
the producer has no content; an empty string is not a declaration. This applies
to recommendation, entity, action, display-unit/template, and Goal-description
copy.

The public Offer is a user-facing projection. `targeting` and
`conversion_rule` belong only to the Partner artifact. Operational status,
audit, priority, resolved supply provenance, eligibility decisions, freshness evaluation,
affiliate parameters, negotiated overrides, fee splits, settlement state, and
refund state remain internal policy. The Partner-only `source_offer_id` is the
one declared source identity and is removed from the public projection.

## Identity and authority

| Field | Normative meaning |
| --- | --- |
| `offer_id` | AON-issued, globally unique inventory identity for one logical Offer. It is stable across dispatches. Partner source ids, database row ids, and request ids must not be serialized into this field. |
| `offer_instance_id` | AON-issued identity for one dispatch of an Offer in a public response. It changes for a later dispatch of the same `offer_id` and is propagated unchanged through click and conversion attribution. |
| `source_offer_id` | Partner-issued stable opaque identity for one logical source Offer within the identity namespace configured for that integration. It exists only in the Partner supply carrier. |
| `version` | Schema-bound Offer payload marker. Its allowed value is fixed by the applicable public or Partner Offer JSON Schema; consumers must not use it for transport negotiation. |

The Partner Offer requires `source_offer_id`. AON resolves `(Partner, identity
namespace, source_offer_id)` to its own canonical `offer_id` before ranking or
public projection. Partner Offers never carry `offer_id`, `offer_instance_id`,
or `match_reason`; AON adds those fields only when it creates a public Query
response projection. Producers must not rotate `source_offer_id` for an
unchanged logical source Offer or reuse it for a different Offer in the same
namespace.

A consumer must preserve `offer_id` as the inventory identity and
`offer_instance_id` as the attribution identity. Neither value may be replaced
with `request_id`, an action URL, a source id, or a click id.

## Language profile

`content_language` and response `language` use the stable-v1.0 BCP-47 profile:

- the primary language subtag has two or three ASCII letters;
- extlang, script, region, variant, extension, and private-use subtags are
  allowed in their BCP-47 order;
- matching is case-insensitive, while producers should use canonical casing;
- labels such as `english` and underscore forms such as `en_US` are invalid;
- extension-bearing tags such as `en-US-u-hc-h12` are valid.
- each extension singleton appears at most once; for example,
  `en-u-ca-gregory-u-nu-latn` is invalid because `u` is repeated.

`content_language` describes Offer copy. Partner `targeting[].language` is a
separate two-letter lowercase ASCII primary-language eligibility constraint.
The targeting field validates this syntax profile only and does not assert ISO
639 registry membership. Neither field selects the protocol version.

## Public Offer content

### Offer information

| Field group | Meaning |
| --- | --- |
| `offer_info.title`, `short_description`, `description` | `title` identifies the Offer; optional `short_description` is compact preview copy, while required `description` is the full user-facing explanation. Neither description field is AON ranking or per-request matching rationale. |
| `offer_info.category.id`, `secondary_category_ids` | Primary and additional AON Taxonomy v1 classifications. Secondary entries must be disjoint from the primary and one another. |
| `offer_info.tags` | Optional descriptive labels. They do not replace taxonomy, eligibility, or targeting. |
| `offer_info.rating.*` | Source-provided rating value on a five-point scale, with optional positive observation count and source. `count` is omitted when no observation count is available; zero is not a meaningful represented sample. It is not an AON endorsement. |
| `offer_info.properties[]` | Structured user-facing facts. `display_pattern` may use only `${type}`, `${value}`, and `${unit}` from the same item. |
| `offer_info.recommendation_reason` | Static Partner/provider-authored Offer copy. It is not per-request reasoning, is not a ranking input, and is unaffected by `thinking_mode`. |
| `offer_info.commercial.*` | Public user price and fulfillment presentation. It is distinct from Goal commission and final settlement. |
| `offer_info.start_at`, `expire_at` | Inclusive user-visible availability bounds. When both are present, `expire_at` must not precede `start_at`. |

When supplied, `offer_info.short_description` remains serialized exactly as
provided but must contain at most 500 Unicode code points. Semantic validation
normalizes it to NFC and trims ECMAScript whitespace for validation only. An
empty normalized value is rejected with `short_description_blank`; otherwise,
the value may contain at most 50 `isWordLike` segments from
`Intl.Segmenter("und", { granularity: "word" })`, or it is rejected with
`short_description_word_limit`. These shared `offer_info` rules apply to both
public Offers and Partner Offers through the shared public Offer schema.

### Offer type values

| Value | Meaning |
| --- | --- |
| `physical_product` | A tangible good delivered, collected, or used physically. |
| `digital_goods` | A purchased or licensed digital item, such as software, an asset, or downloadable media. |
| `content` | Editorial, educational, entertainment, or other consumable content where access to the content is the primary offer. |
| `online_service` | A service primarily delivered through a website, application, API, or other online channel. |
| `offline_service` | A service primarily fulfilled at a physical location or through an offline appointment. |

### Entity type values

| Value | Meaning |
| --- | --- |
| `merchant` | Party that sells or contracts the offered good or service. |
| `brand` | Brand owner or named brand represented by the Offer. |
| `provider` | Party that performs or delivers the offered service. |
| `publisher` | Party that publishes or controls the offered content. |
| `other` | Responsible party that does not fit the preceding roles; producers should prefer a specific value when one applies. |

### Listing source kind values

| Value | Meaning |
| --- | --- |
| `platform` | Intermediary platform that presents or distributes listings but is not necessarily a transaction marketplace. |
| `marketplace` | Multi-seller venue where the listing is presented and a transaction can ordinarily be initiated. |
| `merchant_site` | Site controlled by the merchant responsible for the Offer. |
| `official_site` | Official site controlled by the represented brand, provider, or publisher. |
| `other` | Explicit source outside the preceding classes. |

`entity` identifies the responsible party. `listing_source` identifies where
listing information was observed. `action` identifies what is invoked after the
user chooses the Offer. They are not interchangeable.

## URI policy

Resource URIs (`entity.website`, `entity.logo`, `listing_source.logo`, and
`material[].url`) must be absolute HTTPS URIs, must contain a host, and must not
contain URI userinfo. Non-ASCII components are percent-encoded. Producers omit
an optional resource rather than substituting HTTP, relative, credential-bearing,
or executable content URIs.

`action.payload.url` is executable and follows `action.type`:

| Action type | URI rule |
| --- | --- |
| `open_url` | Absolute HTTPS URI with a host and no userinfo. |
| `deep_link` | Safe absolute application or universal-link URI recognized by the consumer. |
| `open_app` | Safe absolute URI whose purpose is to launch an installed application or application route. |
| `custom` | Safe absolute integration-specific URI; consumers execute it only when they explicitly recognize the scheme and contract. |

`javascript`, `data`, `vbscript`, and `file` schemes are forbidden for every
action type. A consumer must not execute an unrecognized `custom`, `deep_link`,
or `open_app` scheme.

### Consumer action values

| Value | User intent |
| --- | --- |
| `learn_more` | Read more information before committing. |
| `buy` | Purchase a product or service. |
| `book` | Reserve a time, place, stay, ticket, or service. |
| `subscribe` | Start a recurring or continuing subscription. |
| `download` | Retrieve software, content, or another digital asset. |
| `claim` | Accept or redeem an available benefit or offer. |
| `sign_up` | Create an account or register interest. |
| `open` | Open the destination without asserting a more specific user intent. |

`consumer_action` describes the user-facing call to action. It does not identify
a billable conversion event; `goals[].event` does that.

### Destination type values

| Value | Destination surface |
| --- | --- |
| `web` | Browser or web surface. |
| `app` | Native or installed application surface. |
| `phone` | Telephone call or dialer surface. |
| `email` | Email composition or email destination. |

### Material format values

| Value | Meaning |
| --- | --- |
| `image` | Static image resource. |
| `video` | Video resource. |
| `html5` | HTML5 creative package retrieved over HTTPS; its contents remain untrusted and require consumer sandboxing. |

### Claim kind values

| Value | Meaning |
| --- | --- |
| `advertiser_claim` | Statement made by the advertiser, Partner, or responsible entity. |
| `user_benefit` | Source-provided statement of a potential user benefit. |
| `availability` | Source-provided statement about availability, timing, or stock. |

Claims remain source-provided statements and must not be presented as AON
verification or endorsement.

## Public price and Goal commission

`offer_info.commercial.price` is what the user is shown as the price of the
offered good or service. `goals[].pricing` is the gross commission basis that the
Partner declares payable to AON for an approved, attributed conversion. The two
prices are independent.

Goal events are unique within one Offer. Each pricing branch is strictly
positive:

| Model | Basis |
| --- | --- |
| `cpa` | Fixed `amount` and `currency` payable by the Partner to AON for one approved occurrence of `goals[].event`. |
| `cps` | Percentage `rate`, greater than zero and no greater than 100, applied to the gross conversion amount and currency reported by the Partner for one approved sale. |

A CPS conversion cannot be settled from the public declaration alone: the
conversion report must supply a valid gross amount and currency. Pricing is
before AON service fees and developer revenue share. It is not a promise of the
developer's net payout and not a final invoice amount. Conversion validation,
holds, fraud review, refunds, disputes, chargebacks, currency reconciliation,
and adjustments may reduce or reverse final settlement. Those lifecycle events
are not encoded by rewriting the immutable Offer response.

### Public price unit values

| Value | Meaning |
| --- | --- |
| `one_time` | One-time displayed price. |
| `night` | Price per night. |
| `day` | Price per day. |
| `week` | Price per week. |
| `month` | Price per month. |
| `year` | Price per year. |

## Partner targeting

The Partner Offer is a normalized supply/configuration artifact, not an Agent
response. `targeting` and `conversion_rule` are Partner-authored inputs. AON
evaluates them before ranking and strips them from the public Offer projection.

Targeting uses this truth table:

1. Omitted `targeting` means no Partner targeting constraint.
2. A present `targeting` array is non-empty.
3. Every rule contains at least one non-empty declared dimension.
4. Declared dimensions within one rule are combined with AND.
5. Rules within the array are combined with OR; one matching rule makes the
   targeting expression eligible.
6. A dimension omitted from a rule is unconstrained for that rule.
7. Missing, unverified, or unknown user context fails a declared dimension and
   therefore fails that rule. Another rule may still match.
8. A location matches a declared location when it is that full-catalog location
   or a descendant in the resolved ACTIVE hierarchy. `geo.exclude` wins over
   `geo.include`.
9. An omitted `geo.include` means no positive include restriction. A present
   include or exclude list is non-empty.
10. Every declared `location_id` must be a numeric entry in the AON Full
    Location Catalog v1 snapshot pinned by the stable-v1.0 release; numeric
    grammar alone is not sufficient. The catalog contains every legacy AON
    Location Registry v1 country entry, so legacy country targeting semantics
    are unchanged. Unknown and non-numeric values fail closed with
    `location_registry_membership`.

`eligibility.min_age` requires verified age context. Targeting is an eligibility
filter, not a substitute for independent legal, consent, sensitive-category, or
platform compliance gates.

## Partner conversion rule

`conversion_rule` is optional. When the object or one of its members is omitted,
these stable-v1.0 defaults apply:

| Field | Default | Normative behavior |
| --- | --- | --- |
| `click_window_hours` | `720` | A conversion time from the click timestamp through click timestamp plus the window is eligible. `0` disables click attribution. |
| `view_window_hours` | `0` | Same inclusive boundary for a view. `0` disables view-through attribution. |
| `attribution_model` | `last_click` | Select the latest eligible click. `first_click` selects the earliest. A qualifying click always takes precedence over a view; the same earliest/latest rule applies to views only when no click qualifies. |
| `dedup_strategy` | `first` | `first` accepts only the first qualifying distinct business conversion in scope; `all` accepts every distinct qualifying business conversion. |

The deduplication scope is Partner + resolved canonical `offer_id` + Goal event
+ stable business conversion identity. `order_id`, `partner_txn_id`, or
`event_id` supplies that identity when available. Retries of one identity are
idempotent under both strategies and never count twice.

`minimum_amount` is a strictly positive threshold on the gross reported
conversion amount. Because v1.0 has no currency member beside this field, it is
valid only for a single-currency integration whose fixed conversion currency is
declared out of band. Multi-currency integrations must omit this field and keep
currency-qualified thresholds in internal Partner policy.

### Attribution model values

| Value | Meaning |
| --- | --- |
| `last_click` | Select the latest eligible interaction in the applicable click-first source class. |
| `first_click` | Select the earliest eligible interaction in the applicable click-first source class. |

### Deduplication strategy values

| Value | Meaning |
| --- | --- |
| `first` | Only the first qualifying distinct conversion in the deduplication scope is billable. |
| `all` | Every distinct qualifying conversion in scope is billable; retries of an existing identity remain idempotent. |

## Query response

| Field | Normative meaning |
| --- | --- |
| `request_id` | Correlation id shared with the current Query request. |
| `protocol_version` | Exact transport protocol selected by the request header and echoed in the response body (`"1.0"`); it is independent of Offer payload fields. |
| `language` | Selected language of user-facing response content under the stable-v1.0 language profile. |
| `offers` | Public Offer projections in descending selection priority for this request. Order is meaningful within one response but is neither stable nor comparable across requests. |
| `engagement.refinements[]` | Suggestions that narrow the current intent. |
| `engagement.followup_topics[]` | Adjacent-topic suggestions ordered by descending `confidence`. |
| `query_helper.request_patch` | Non-null, non-destructive partial update for a subsequent Query. |
| `hooks[]` | Change cues comparing one returned Offer with one explicit previous response baseline. They are not watch registrations or delivery guarantees. |

### Query Helper update profile

`request_patch` is not an unrestricted RFC 7386 merge patch. It is a constrained
partial update limited to `intent.signals`, `constraints.category_ids`, and
`constraints.excluded_category_ids`:

- omission preserves the existing field;
- every supplied nested object contains at least one actual update; empty
  `intent`, `signals`, and `constraints` objects are invalid;
- objects merge recursively;
- arrays replace the previous array;
- an empty category-constraint array is a meaningful replacement that clears
  that constraint list;
- `null` is invalid and cannot remove a field;
- explicit current-turn user conditions win over a conflicting suggested value;
- the merged Query is validated before use.

### Follow-up basis values

| Value | Meaning |
| --- | --- |
| `category_complement` | Related category that complements the current category. |
| `sequential_journey` | Likely next step in the same user journey. |
| `problem_to_product` | Product or service direction derived from the problem expressed by the user. |
| `comparison_alternative` | Alternative suitable for comparison with the current direction. |
| `user_interest` | Adjacent direction supported by explicit current or bounded session interest. |
| `seasonal` | Direction whose relevance comes from a current season or time-bound occasion. |

`confidence` is a producer-normalized relevance estimate, not a calibrated
probability. Values are comparable only among `followup_topics` in the same
response from the same producer and are ordered descending. They must not be
compared across requests, producers, or protocol versions.

### Empty result values and precedence

`empty_reason` is required exactly when `offers` is empty. If multiple causes
apply, the producer selects the first applicable value in this precedence:

| Precedence | Value | Meaning |
| ---: | --- | --- |
| 1 | `consent_missing` | Required user consent is absent, so otherwise eligible Offer processing cannot proceed. |
| 2 | `scene_suppressed` | The current product scene or policy suppresses Offer presentation. |
| 3 | `frequency_capped` | Candidate presentation is blocked by an applicable frequency cap. |
| 4 | `no_material` | Candidates exist but none has a usable action or presentation resource required by the current surface. |
| 5 | `below_relevance_threshold` | No remaining eligible candidate meets the response relevance threshold. |

### Hook values and baseline

Every Hook requires:

- `subject_offer_id`, which references an Offer returned in the same response;
- `baseline_request_id`, which equals the current request's
  `context.session.previous_request_id`; and
- current evidence that the named field class differs from that baseline.

| Hook kind | Meaning |
| --- | --- |
| `price_change` | Public commercial price or price unit changed. |
| `availability_change` | User-visible availability bounds or availability claim changed. |
| `eligibility_change` | The Offer's eligibility outcome for the current context changed; private targeting details remain undisclosed. |
| `content_change` | Material user-facing title, description, action copy, claim, or creative content changed. |

A Hook does not create a subscription, claim future monitoring, or guarantee
notification delivery.

## Change control

Adding or changing a public field meaning requires an RFC-governed protocol
change. Every new `properties` entry must include a non-empty normative Schema
description, update this reference when its field group or enum family changes,
and pass the field-semantics coverage test before publication.
