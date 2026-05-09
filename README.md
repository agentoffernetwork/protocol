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

## Choose Your Path

| If you want to... | Start here | Then go to |
|-------------------|------------|------------|
| Query offers from an agent or app | [Query API](specs/query-api.md) | [Examples](https://github.com/agentoffernetwork/examples/blob/main/http/offer-query-request.json), [Schema](https://github.com/agentoffernetwork/schema/blob/main/json-schema/offer-query-schema-v0.1.json) |
| Understand the canonical offer object | [Offer Schema](specs/offer-schema.md) | [Schema](https://github.com/agentoffernetwork/schema/blob/main/json-schema/offer-schema-v0.1.json), [Examples](https://github.com/agentoffernetwork/examples) |
| Choose a category or sub-type | [Category Taxonomy](specs/category-taxonomy.md) | [Offer Schema category attributes](specs/offer-schema.md#category-attributes) |
| Track clicks and conversions | [Events](specs/events.md) | [Postback](specs/postback.md) |
| Check whether a field is canonical or historical | [Contract Governance](specs/contract-governance.md) | [RFC process](https://github.com/agentoffernetwork/rfcs) |
| Propose a protocol change | [RFCs](https://github.com/agentoffernetwork/rfcs) | [CONTRIBUTING.md](CONTRIBUTING.md) |

## Quick Start

1. Read the [Query API](specs/query-api.md) if you are building an agent, app, or SDK that searches for offers.
2. Read the [Offer Schema](specs/offer-schema.md) to understand the `offers[]` objects returned by the Query API.
3. Use [Schema](https://github.com/agentoffernetwork/schema) to validate request/response payloads.
4. Browse [Examples](https://github.com/agentoffernetwork/examples) for copyable JSON payloads.
5. Read [Contract Governance](specs/contract-governance.md) when you need field lifecycle, source references, or stale-field handling.
6. Open an [RFC](https://github.com/agentoffernetwork/rfcs) before proposing field, enum, compatibility, or governance changes.

## Implementation-Ready Core

| Specification | Description |
|---------------|-------------|
| [Offer Schema](specs/offer-schema.md) | Canonical offer object with 11 category types, 70+ sub-types, and per-vertical attribute contracts |
| [Query API](specs/query-api.md) | `POST /v1/offers/query` with multimodal intent, structured filters, and pagination |
| [Events](specs/events.md) | Click and conversion event definitions with stable identifiers for full-funnel attribution |
| [OfferProvider API](specs/offer-provider-api.md) | Partner-facing request/response, signing, and error semantics for standardized offer supply |
| [Postback](specs/postback.md) | AON→Agent and Partner→AON callback rules, payloads, signatures, and retry behavior |
| [Contract Governance](specs/contract-governance.md) | Field lifecycle, source references, public follow rules, and stale-field handling |

These documents make up the current **implementation-ready core** of the protocol. They
are still published as **v0.1 Draft** because the ecosystem and governance process are
young, but the core contract surface is intended for real integration and feedback.

## First Query in 5 Minutes

For most agent integrations, the shortest useful path is:

1. Open [`specs/query-api.md`](specs/query-api.md).
2. Copy the minimal `POST /v1/offers/query` request.
3. Send `context.user_profile` and `intent.content[]`.
4. Read `request_id` and `offers[]` in the response.
5. Use `offers[].offer_instance_id` when reporting downstream click or conversion attribution.

The Query API page links to the machine-readable schema and canonical examples so you can move from prose to validation without hunting through repositories.

## Companion Drafts

The following documents are published for direction-setting and early review, but they are
**not** part of the current implementation-ready core:

| Specification | Current role |
|---------------|--------------|
| [Agent Identity](specs/agent-identity.md) | Informational companion spec for future registration and attribution identity work |
| [Compliance Guide](specs/compliance-guide.md) | Informational companion spec for future machine-readable compliance extensions |

### Category Types

The current public category surface is defined in
[`specs/category-taxonomy.md`](specs/category-taxonomy.md).

- Current canonical output contains 11 categories in v0.1.
- Historical aliases may still be accepted in input normalization layers, but they are not canonical public output.
- External taxonomy mappings are intentionally handled outside the current implementation-ready core.

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
    category-taxonomy.md    # Canonical category registry and boundary rules
    offer-schema.md        # Canonical offer object definition
    query-api.md           # Offer discovery API
    events.md              # Click & conversion tracking
    contract-governance.md # Field lifecycle and public follow rules
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
| [`agentoffernetwork/examples`](https://github.com/agentoffernetwork/examples) | Canonical request/response payloads aligned with the current public category surface |
| [`agentoffernetwork/rfcs`](https://github.com/agentoffernetwork/rfcs) | Protocol change proposals and governance |

### Source-of-Truth Roles

| Surface | Role |
|---------|------|
| This `protocol` repo | Human-readable contract and governance source |
| `schema` repo | Machine-readable JSON Schema and TypeScript types |
| `examples` repo | Canonical request/response payloads for inspection and validation |
| `rfcs` repo | Required path for semantic contract changes |

Markdown explains the contract. Schema and examples make it verifiable. RFCs govern semantic change.

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
