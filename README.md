# AgentOffer Protocol

AgentOffer Protocol is the open protocol for AI agent commerce.

This repository contains the canonical, human-readable specification for describing offers, querying offers, tracking attribution events, identifying agents, and handling compliance disclosures in agent-driven recommendation flows.

## Why This Exists

AI agents need a shared way to describe commercial offers, request relevant recommendations, and report click and conversion outcomes. AgentOffer Protocol provides a common vocabulary so protocol authors, SDK developers, service builders, and ecosystem partners can interoperate on the same contract.

## Included In v0.1

- Offer Schema
- Offer Query API
- Click and Conversion Events
- Agent Identity
- Compliance and Disclosure Guide

## Start Here

1. Read `specs/offer-schema.md`
2. Read `specs/query-api.md`
3. Read `specs/events.md`
4. Review the related `schema`, `examples`, and `rfcs` repositories

## Repository Map

- `specs/` normative protocol documents
- `.github/` community templates

## Current Status

- Version: `v0.1`
- Status: `Draft`

## Related Repositories

- `agentoffernetwork/schema` for machine-readable contracts
- `agentoffernetwork/examples` for request/response and integration examples
- `agentoffernetwork/rfcs` for protocol change proposals and governance

## Contributing

- Open documentation fixes directly as pull requests when the change is editorial and non-breaking
- Open issues for questions, bugs, and clarity gaps
- Open an RFC before changing protocol semantics, field meanings, or governance rules

See `CONTRIBUTING.md` for contribution routing details.
