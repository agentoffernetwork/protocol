# Changelog

All notable changes to the AgentOffer Protocol specification will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added

- AON Taxonomy v1 category registry and migration guidance, replacing the
  draft v0.1 `category.type + attributes.sub_type` main path with
  `offer_info.category.id`.
- Added Contract Governance spec covering field lifecycle, source references, public follow rules, stale-field denylist, and compatibility allowlist.
- Offer targeting `os` dimension (`ios`/`android`/`windows`/`macos`/`linux`) and Query `user_profile.country` (ISO 3166-1 alpha-2) for geo/OS targeting; documented intra-rule AND / inter-rule OR matching semantics in `offer-schema.md` (SVC-CORE-F024, non-breaking).
- `bid.model_subtype` (CPA Type): optional free-form token (`^[A-Za-z0-9_-]{1,16}$`) qualifying the CPA bid model; common values Registration / Submission / Transaction / Retention / Install; partners may define custom values (WS-002).
- Canonical location targeting via AON Location Registry v1 `location_id`
  values, sourced from Google Ads Geo Target Criteria IDs and limited in the
  first release to `COUNTRY`, `REGION`, and `CITY`.
- Non-PII age threshold targeting via Query
  `context.user_profile.verified_age_over` and Offer
  `targeting[].eligibility.min_age`.
- Optional `offer_info.tags` for partner-supplied content matching hints on
  returned Offer payloads.

### Changed

- Query API and OfferProvider category constraints now use
  `constraints.category_ids` with AON Taxonomy v1 subtree matching semantics.
- Narrowed `bid.model` enum to `cpa` / `cps` / `hybrid` (WS-002).
- Geo targeting semantics now use self-or-ancestor `location_ids`, fail closed
  on unknown locations, and make `geo.exclude` override `geo.include`; legacy
  country strings remain migration-compatible.
- AON Location Registry v1 exposes `parent_location_id` as the hierarchy source
  and no longer publishes a separate ancestor cache.

### Removed

- `bid.model` values `cpl` (cost per lead) and `cpi` (cost per install); represent lead/install as `cpa` + `model_subtype` (`Submission` / `Install`) instead. Adapters (Impact/MobPower) and stored offers are migrated losslessly (WS-002).

## [0.1.0] - 2026-03-28

### Added

- Offer Schema specification with 6 category types and 40+ sub-types
- Query API specification (`POST /v1/offers/query`) with multimodal intent support
- Events specification (click and conversion tracking with attribution rules)
- Agent Identity specification (minimal registration model)
- Compliance Guide (disclosure requirements and restriction policies)
- RFC 2119 requirement levels (REQUIRED / RECOMMENDED / OPTIONAL)
- Bid models: CPA, CPS, CPL, CPI, Hybrid
- Sub-ID tracking for granular attribution
- Conversion types: sale, lead, install, subscription, trial, custom

## [0.1.1] - 2026-04-23

### Changed

- Promoted `health_beauty`, `fashion`, `food_grocery`, `home_garden`, and `automotive` from reserved/future to canonical public categories
- Expanded the canonical category surface from 6 to 11
- Added common attribute definitions for the 5 newly canonical categories

### Status

- Version: `v0.1`
- Status: `Draft`
