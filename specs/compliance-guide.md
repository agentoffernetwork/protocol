# Compliance and Disclosure Guide v0.1

**Version**: 0.1
**Status**: Draft — Placeholder
**Last Updated**: 2026-03-25

> **Note**: This specification is a forward-looking placeholder. The `compliance` fields referenced below (e.g., `compliance.disclosure_required`, `compliance.terms_url`) are **not yet part of the v0.1 Offer Schema**. They represent planned extensions for a future protocol revision. The behavioral guidance in this document is informational and does not impose schema-level requirements at this time.

## Introduction

This guide defines the minimum disclosure expectations for agents that recommend monetized offers through AgentOffer Protocol. The objective is to make promotional intent clear to end users and to support responsible affiliate-style recommendations from the start.

When implemented, this guide will be supported by a `compliance` object in the Offer Schema with fields such as:

- `compliance.disclosure_required`
- `compliance.disclosure_text`
- `compliance.terms_url`
- `compliance.restricted_claims`

## Disclosure Requirements

- Agents must clearly disclose when a recommendation is sponsored, monetized, or affiliate-linked.
- The disclosure must appear close to the recommendation itself and not be hidden in unrelated settings or footnotes.
- Disclosures should remain understandable to a normal end user without legal interpretation.
- When an Offer object provides `compliance.disclosure_required = true`, agents must render a disclosure.
- When an Offer object provides `compliance.disclosure_text`, agents should use that text or a stricter approved equivalent.

## Disclosure Templates

### Short Form

`[Ad] This is an affiliate recommendation.`

### Long Form

`Disclosure: I may earn a commission if you purchase through this link, at no additional cost to you.`

## Example Usage

An agent recommendation may end with one of the approved templates when presenting a tracked offer link. The short form is appropriate for compact UI surfaces. The long form is recommended when the interface provides more room or when legal review prefers fuller language.

If an offer includes `terms_url`, products may link to it in expanded offer details or terms views. If an offer includes `restricted_claims`, agent implementations must suppress or rewrite recommendation copy that would violate those restrictions.

## Prohibited Behaviors

- Hiding or omitting the promotional nature of a recommendation.
- Falsely claiming neutrality when the agent or developer may receive compensation.
- Presenting fabricated performance claims, discounts, or guarantees that are not supported by the advertiser.

## Regulatory Reference

This v0.1 guide is aligned with the spirit of U.S. FTC endorsement and advertising disclosure expectations. Teams implementing the protocol should review the latest FTC guidance and any local-market advertising rules before production launch.

## Design Decisions

- v0.1 focuses on practical disclosure language rather than a full legal framework so developers can adopt the protocol quickly.
- Two template lengths are provided because recommendation surfaces vary across chat, voice, cards, and workflow systems.
- The guide is written as a protocol companion, not legal advice; product teams remain responsible for market-specific compliance.
- Offer-level `compliance` fields are treated as the machine-readable source of truth, while this guide provides the human-readable policy and usage rules around them.

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-03-20 | Initial draft. |
| 0.1 | 2026-03-20 | Added explicit mapping to the reference Offer Schema `compliance` object and its execution rules. |
