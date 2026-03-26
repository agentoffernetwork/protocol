# AgentOffer Protocol

AgentOffer Protocol is the open standard for AI agent offer exchange.

This repository contains the canonical, human-readable specification for describing offers, discovering them, tracking attribution events, identifying agents, and handling compliance disclosures in agent-driven recommendation flows.

## Why This Exists

AI agents need a shared way to describe commercial offers, request relevant recommendations, and report outcomes. AgentOffer Protocol provides a common vocabulary so protocol authors, SDK developers, service builders, and ecosystem partners can interoperate on the same contract.

## What's In v0.1

| Spec | Description |
|------|-------------|
| **Offer Schema** | Canonical offer object with 6 category types, 40+ sub-types, and per-type attribute contracts |
| **Query API** | `POST /v1/offers/query` with multimodal intent, user context, and pagination |
| **Events** | Click and conversion event definitions with stable identifiers for attribution |
| **Agent Identity** | Minimal registration model with agent_id, developer linkage, and API key binding |
| **Compliance Guide** | Machine-readable disclosure requirements and restriction policies |

### Category Types

The protocol defines 6 top-level industry verticals, each with typed `attributes` and `sub_type` discrimination:

- `software_saas` — SaaS / subscription software
- `travel_hospitality` — hotels, flights, car rentals
- `education` — online courses, certifications, bootcamps
- `financial_service` — credit cards, insurance, loans
- `electronics` — smartphones, laptops, audio, wearables
- `entertainment` — games, streaming, AI companions, sports betting, live streaming

### Requirement Levels

Field requirements follow [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119):

| Keyword | Meaning |
|---------|---------|
| **REQUIRED** | Field MUST be present with a valid, non-empty value |
| **RECOMMENDED** | Field SHOULD be present and follow the standard structure; value MAY be empty |
| **OPTIONAL** | Field MAY be omitted entirely |

## Start Here

1. Read `specs/offer-schema.md`
2. Read `specs/query-api.md`
3. Read `specs/events.md`
4. Review the related `schema`, `examples`, and `rfcs` repositories

## Repository Map

- `specs/` — normative protocol documents
- `.github/` — community templates

## Current Status

- Version: `v0.1`
- Status: `Draft`

## Related Repositories

| Repository | Purpose |
|------------|---------|
| [`agentoffernetwork/schema`](https://github.com/agentoffernetwork/schema) | Machine-readable JSON Schema, TypeScript types, and validators |
| [`agentoffernetwork/examples`](https://github.com/agentoffernetwork/examples) | Request/response payloads covering all 6 category types |
| [`agentoffernetwork/rfcs`](https://github.com/agentoffernetwork/rfcs) | Protocol change proposals and governance |

## Contributing

- Open documentation fixes directly as pull requests when the change is editorial and non-breaking
- Open issues for questions, bugs, and clarity gaps
- Open an RFC before changing protocol semantics, field meanings, or governance rules

See `CONTRIBUTING.md` for contribution routing details.
