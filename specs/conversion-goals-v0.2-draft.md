# Conversion Goals v0.2 Draft (Historical / Superseded)

**Version**: 0.2 draft (rev. 2)
**Status**: Historical / superseded by [`offer-schema-v0.2.md`](./offer-schema-v0.2.md)
**Last Updated**: 2026-07-09

## Introduction

This document defines the non-GA v0.2 draft overlay for expressing conversion goals in AgentOffer Protocol.

A conversion goal is the smallest billable statement an offer can make: *one recognized occurrence of an event, at a stated price*. The draft keeps the protocol surface minimal on purpose — identity, matching machinery, vocabularies, and migration history all live outside the base contract.

The active v0.1 Offer Schema remains compatible. The active `offer-schema-v0.1.json` still requires its top-level pricing field; this draft does not remove, weaken, or replace the stable v0.1 contract.

## Scope

The draft introduces `goals[]` as the native expression for billable conversion goals on one offer. Each goal contains exactly an event, a pricing object, and an optional advisory description. Runtime APIs, settlement execution, event-name mapping, UI configuration, and database migrations are platform concerns and are not defined here.

## Goals Shape

| Field | Level | Description |
|-------|-------|-------------|
| `goals` | REQUIRED | Array of conversion goals. MUST contain at least one item. Array order carries no semantics. The protocol sets no upper bound on the array; per-offer goal ceilings are platform policy. |
| `goals[].event` | REQUIRED | Semantic identity and reference key of the goal. Lowercase slug matching `^[a-z0-9][a-z0-9_.-]{0,63}$`. |
| `goals[].pricing` | REQUIRED | Price paid for one recognized occurrence of the event. Discriminated union of `cpa` and `cps`. |
| `goals[].description` | OPTIONAL | Advisory human-readable note, at most 500 characters. |

Pricing branches:

| Branch | Required fields | Meaning |
|--------|----------------|---------|
| `cpa` | `model`, `amount`, `currency` | Fixed payout per recognized occurrence. `amount` is a positive decimal string (at most 12 integer and 6 fractional digits); `currency` is ISO 4217. |
| `cps` | `model`, `rate` | Percentage of the conversion value. `rate` is a positive decimal string, at most 4 fractional digits, structurally capped at 100. A ratio carries no currency. |

Each pricing branch pins `model` with a JSON Schema `const` and rejects unknown properties.

## Normative rules

1. **Event uniqueness.** `goals[].event` values MUST be unique within one offer (exact string comparison). The protocol defines no vocabulary for `event`; only the syntax is constrained. No canonicalization is applied: values differing only by separator placement (for example `install` and `install.`) are distinct events, and producers SHOULD avoid trailing or consecutive separators.
2. **Replacement semantics.** Changing a goal's `event` replaces the goal (delete + create). It is not a rename: consumers MUST treat the previous goal as removed. Cross-message references to a goal use `(offer_id, event)`.
3. **Positive exact amounts.** `pricing.amount` and `pricing.rate` MUST be greater than zero. Values are exact decimal-string passthrough: there is no implicit rounding, scale normalization, percentage conversion, currency-minor-unit conversion, or trailing-zero trimming. Producers that cannot express the value within the stated precision MUST reject the payload instead of silently correcting it.
4. **Recognition semantics.** A goal is completed by a recognized occurrence of its event. Recognition rules — attribution windows, deduplication, minimum thresholds, review, refund reversal — belong to the offer-level `conversion_rule` of the active v0.1 schema and to platform policy; they are not part of the goal.
5. **Advisory fields.** Consumers MUST NOT derive billing, matching, or settlement behavior from `description` or any other advisory field. Absence or inaccuracy of advisory content does not affect the contract.

## Events and postback boundary

This v0.2 draft does not add per-goal event fields to the active v0.1 Click and Conversion Events specification. v0.1 events/postback remain flat compatibility: conversion events keep their existing flat fields, and postback payload shape remains governed by `events.md` and `postback.md`.

Per-goal conversion representation, settlement attribution, payout item references, and any public event fields that reference goals are deferred to WS-15-S6 as a conformance follow-up. Downstream services MUST NOT infer a new public event shape from this draft without that follow-up.

## Appendix A (non-normative) · Well-known event names

The protocol constrains only the syntax of `event`. For interoperability, the following well-known event names are recommended when they fit the offer's semantics. This list is advisory and not required for validation: schema validation never depends on this appendix or on the companion data file (`schema/conversion-events/aon-conversion-events-v0.2-draft.json`).

| Well-known name | Typical meaning |
|-----------------|-----------------|
| `install` | App install or first open. |
| `sale` | Paid purchase or order. |
| `lead` | Lead form, registration, or qualified inquiry. |
| `subscription` | Subscription start or conversion. |
| `trial` | Trial start or trial-like activation. |

Platforms MAY provide display labels, localization, classification, or historical statistics for any event name as enrichment outside the protocol document.

## Appendix B (non-normative) · Migration from v0.1

This appendix is a migration reference for implementers. It is non-normative: the v0.2 draft schema contains no compatibility fields, and any v0.1-compatible output surface is derived by the serving platform, not authored in payloads.

| v0.1 surface | v0.2 draft mapping |
|--------------|--------------------|
| Top-level fixed-price pricing (`cpa`) | Maps directly to a goal with `pricing.model = "cpa"`, same `amount` and `currency`. |
| Top-level revenue-share pricing (`cps`) | The v0.1 percentage value maps to `pricing.rate`. The v0.1 currency field has no v0.2 target and is dropped: a ratio carries no currency. |
| Top-level `hybrid` pricing | No v0.2 equivalent. Offers using v0.1 hybrid pricing MUST be restructured manually into one or more `cpa`/`cps` goals before adopting the draft. |
| `conversion_rule.accepted_types` | Superseded. The set of billable events of an offer is exactly `goals[].event`. |

Serving platforms that must keep emitting a v0.1-compatible pricing view to legacy consumers derive it from `goals[]` at the serving layer; this derivation is a platform responsibility and is outside the protocol contract.

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.2 draft | 2026-07-09 | Initial non-GA conversion goals draft overlay. |
| 0.2 draft (rev. 2) | 2026-07-09 | Goal minimized to event + pricing (+ advisory description). Removed goal identity field, alias lists, custom-event objects, normative event vocabulary, pricing compatibility fields, and top-level compatibility surfaces; matching and vocabularies moved to platform scope; migration guidance demoted to a non-normative appendix. |
