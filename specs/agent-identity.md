# Agent Identity v0.1

**Version**: 0.1
**Status**: Draft
**Last Updated**: 2026-03-20

## Introduction

This document defines the minimum identity model required to register and recognize an AI agent within AgentOffer Protocol v0.1. The goal is to support attribution, administration, and safe platform operations without introducing unnecessary complexity in the first protocol release.

Unlike the Offer object, agent identity is not modeled as a standalone schema in the current canonical schema package. This document therefore acts as a complementary protocol spec. Its main alignment point with the Offer Schema is the repeated use of `agent_id` in source tracking templates and event attribution flows.

## Agent Registration Fields

| Field | Type | Required | Description |
|------|------|----------|-------------|
| `agent_id` | string | Yes | Platform-assigned unique identifier for the agent. |
| `name` | string | Yes | Human-readable agent name. |
| `type` | enum | Yes | Agent classification: `chatbot`, `assistant`, `workflow`, or `other`. |
| `platform` | string | Yes | Execution environment or provider such as `OpenAI`, `Claude`, or `Custom`. |
| `developer_id` | string | Yes | Identifier of the developer account that owns the agent. |
| `registered_at` | string | Yes | ISO 8601 datetime when the agent was registered. |

## Agent Types

| Value | Meaning |
|------|---------|
| `chatbot` | Conversational bot that directly interacts with end users. |
| `assistant` | Embedded or task-oriented assistant integrated into another product. |
| `workflow` | Automated agent that runs steps or orchestration without a persistent chat UI. |
| `other` | Any agent type that does not fit the current core categories. |

## Authentication Model

- Authentication uses API keys in v0.1.
- An API key is bound to a developer account and associated with one or more registered agents.
- One developer may own multiple agents.
- `agent_id` is assigned by the platform and must not be chosen arbitrarily by the developer.
- This model is compatible with source templates such as `{agent_id}` in `source.tracking_url_template`.

## Design Decisions

- The v0.1 identity model is intentionally minimal because the immediate need is attribution and administrative linkage, not a full trust framework.
- `type` is modeled as a small enum to support analytics and policy enforcement while keeping onboarding simple.
- Platform names remain free-form strings in v0.1 because the ecosystem is evolving quickly and a locked enum would create unnecessary churn.
- API keys are sufficient for the first release; more advanced delegated auth can be introduced in future protocol versions.
- This spec is intentionally separate from the Offer Schema because agent registration belongs to platform identity and attribution, not catalog data.

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-03-20 | Initial draft. |
| 0.1 | 2026-03-20 | Added explicit note that this document complements, rather than duplicates, the reference Offer Schema. |
