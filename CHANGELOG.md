# Changelog

All notable changes to the AgentOffer Protocol specification will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Added

- Added Contract Governance spec covering field lifecycle, source references, public follow rules, stale-field denylist, and compatibility allowlist.
- Offer targeting `os` dimension (`ios`/`android`/`windows`/`macos`/`linux`) and Query `user_profile.country` (ISO 3166-1 alpha-2) for geo/OS targeting; documented intra-rule AND / inter-rule OR matching semantics in `offer-schema.md` (SVC-CORE-F024, non-breaking).

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
