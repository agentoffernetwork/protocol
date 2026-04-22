<p align="center">
  <h1 align="center">AgentOffer Protocol</h1>
  <p align="center">
    The open standard for AI agent offer exchange.
    <br />
    <em>MCP connects agents to tools. A2A connects agents to agents. AON connects agents to revenue.</em>
  </p>
</p>

<p align="center">
  <a href="https://creativecommons.org/licenses/by/4.0/"><img src="https://img.shields.io/badge/license-CC--BY--4.0-blue.svg" alt="License" /></a>
  <a href="#current-status"><img src="https://img.shields.io/badge/version-v0.1-orange.svg" alt="Version" /></a>
  <a href="#current-status"><img src="https://img.shields.io/badge/status-Draft-yellow.svg" alt="Status" /></a>
  <a href="https://github.com/agentoffernetwork/protocol/issues"><img src="https://img.shields.io/github/issues/agentoffernetwork/protocol.svg" alt="Issues" /></a>
  <a href="https://agentoffernetwork.org"><img src="https://img.shields.io/badge/website-https%3A%2F%2Fagentoffernetwork.org-green.svg" alt="Website" /></a>
  <a href="https://github.com/agentoffernetwork/protocol/actions/workflows/lint.yml"><img src="https://github.com/agentoffernetwork/protocol/actions/workflows/lint.yml/badge.svg" alt="Lint" /></a>
</p>

---

## Why This Exists

AI agents are becoming the primary interface for purchase decisions, but agent developers lack a standardized way to monetize recommendations. Traditional affiliate infrastructure was built for websites and content creators -- not for conversational AI agents that reason, compare, and recommend.

**AgentOffer Protocol** provides a shared vocabulary for describing commercial offers, discovering relevant recommendations, and reporting attribution events -- so that agents, platforms, and offer providers can interoperate on one open standard.

```text
+------------------+     +-----------------+     +-------------------+
|  Agent (Claude,  |     |   AON Protocol  |     |  Offer Providers  |
|  GPT, LangChain, | <-> |  Query + Track  | <-> |  (SaaS, Travel,   |
|  CrewAI, ...)    |     |   + Attribute   |     |   Finance, ...)   |
+------------------+     +-----------------+     +-------------------+
```

## Quick Start

1. Read the [Offer Schema](specs/offer-schema.md) to understand the canonical offer object
2. Read the [Query API](specs/query-api.md) to learn how agents discover offers
3. Read the [Events](specs/events.md) spec for click & conversion tracking
4. Browse [Examples](https://github.com/agentoffernetwork/examples) and use the SDK installation pointers in the Developer Tools section below

## Implementation-Ready Core

| Specification | Description |
|---------------|-------------|
| [Offer Schema](specs/offer-schema.md) | Canonical offer object with 6 category types, 40+ sub-types, and per-vertical attribute contracts |
| [Query API](specs/query-api.md) | `POST /v1/offers/query` with multimodal intent, structured filters, and pagination |
| [Events](specs/events.md) | Click and conversion event definitions with stable identifiers for full-funnel attribution |
| [OfferProvider API](specs/offer-provider-api.md) | Partner-facing request/response, signing, and error semantics for standardized offer supply |
| [Postback](specs/postback.md) | AON→Agent and Partner→AON callback rules, payloads, signatures, and retry behavior |

These documents make up the current **implementation-ready core** of the protocol. They
are still published as **v0.1 Draft** because the ecosystem and governance process are
young, but the core contract surface is intended for real integration and feedback.

## Companion Drafts

The following documents are published for direction-setting and early review, but they are
**not** part of the current implementation-ready core:

| Specification | Current role |
|---------------|--------------|
| [Agent Identity](specs/agent-identity.md) | Informational companion spec for future registration and attribution identity work |
| [Compliance Guide](specs/compliance-guide.md) | Informational companion spec for future machine-readable compliance extensions |

### Category Types

Six industry verticals with typed attributes and sub-type discrimination:

| Category | Examples |
|----------|----------|
| `software_saas` | Project management, CRM, dev tools, AI tools |
| `travel_hospitality` | Hotels, flights, car rentals, vacation packages |
| `education` | Online courses, certifications, bootcamps |
| `financial_service` | Credit cards, insurance, loans, investment |
| `electronics` | Smartphones, laptops, audio, wearables |
| `entertainment` | Games, streaming, AI companions, live streaming |

### Bid Models

| Model | Description |
|-------|-------------|
| CPA | Cost per action (fixed amount) |
| CPS | Cost per sale (percentage) |
| CPL | Cost per lead |
| CPI | Cost per install |
| Hybrid | Combined fixed + percentage |

### Requirement Levels

Field requirements follow [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119):

- **REQUIRED** -- field MUST be present with a valid, non-empty value
- **RECOMMENDED** -- field SHOULD be present; value MAY be empty
- **OPTIONAL** -- field MAY be omitted entirely

## How AON Fits the Agent Protocol Stack

| Layer | Protocol | Purpose |
|-------|----------|---------|
| Tool Access | [MCP](https://modelcontextprotocol.io/) | Connect agents to tools and data sources |
| Agent Collaboration | [A2A](https://github.com/google/A2A) | Agent-to-agent task delegation |
| **Agent Monetization** | **AON** | **Agent-to-offer discovery, attribution, and revenue** |

AON is complementary to MCP and A2A. It adds the commercial layer that enables agents to earn revenue from recommendations.

## Repository Map

```text
protocol/
  specs/
    offer-schema.md        # Canonical offer object definition
    query-api.md           # Offer discovery API
    events.md              # Click & conversion tracking
    agent-identity.md      # Agent registration model
    compliance-guide.md    # Disclosure requirements
  .github/                 # Community templates
  CHANGELOG.md             # Version history
  SECURITY.md              # Vulnerability reporting
```

## Ecosystem

| Repository | Purpose |
|------------|---------|
| [`agentoffernetwork/protocol`](https://github.com/agentoffernetwork/protocol) | Human-readable specification (this repo) |
| [`agentoffernetwork/schema`](https://github.com/agentoffernetwork/schema) | JSON Schema, TypeScript types, and validators |
| [`agentoffernetwork/examples`](https://github.com/agentoffernetwork/examples) | Request/response payloads for all 6 categories |
| [`agentoffernetwork/rfcs`](https://github.com/agentoffernetwork/rfcs) | Protocol change proposals and governance |

**Developer Tools:**

| Package | Install | Description |
|---------|---------|-------------|
| TypeScript SDK | `npm install @agentoffernetwork/sdk` | Intent-driven offer search, click tracking, formatting |
| Python SDK | `pip install agentoffernetwork` | Full parity with TypeScript SDK |
| MCP Skill | `aon skill install @agentoffernetwork/skill` | Zero-code integration for Claude and other MCP-compatible hosts |
| ChatGPT Action | See `sdk/chatgpt-action/` | Custom GPT Actions integration for ChatGPT |

## Current Status

- **Version:** `v0.1`
- **Status:** `Draft`
- **Release posture:** `Public beta for the core contract surface`
- **Stability:** Fields marked REQUIRED are unlikely to change. RECOMMENDED and OPTIONAL fields may evolve.
- **Scope note:** Agent Identity and Compliance remain companion drafts and are not yet reflected as standalone schema artifacts in the v0.1 machine-readable package.

## Contributing

We welcome contributions of all kinds:

- **Editorial fixes** -- open a PR directly
- **Questions & bugs** -- open an issue
- **Breaking changes** -- open an [RFC](https://github.com/agentoffernetwork/rfcs) first

See [CONTRIBUTING.md](CONTRIBUTING.md) for detailed routing guidelines.

## Community

- **Website:** [https://agentoffernetwork.org](https://agentoffernetwork.org)
- **Email:** [info@agentoffernetwork.com](mailto:info@agentoffernetwork.com)
- **Security:** [SECURITY.md](SECURITY.md)

## License

This specification is licensed under [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/).
