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

AI agents are becoming the primary interface for purchase decisions, but agent developers lack a standardized way to monetize recommendations. Traditional affiliate infrastructure was built for websites and content creators, not for conversational AI agents that reason, compare, and recommend.

**AgentOffer Protocol** provides a shared vocabulary for describing commercial offers, discovering relevant recommendations, and reporting attribution events so agents, platforms, and offer providers can interoperate on one open standard.

```text
+------------------+     +-----------------+     +-------------------+
|  Agent (Claude,  |     |   AON Protocol  |     |  Offer Providers  |
|  GPT, LangChain, | <-> |  Query + Track  | <-> |  (SaaS, Travel,   |
|  CrewAI, ...)    |     |   + Attribute   |     |   Finance, ...)   |
+------------------+     +-----------------+     +-------------------+
```

## Current v0.1 Snapshot

| Surface | Current contract |
|---------|------------------|
| Agent-facing discovery | `POST /v1/offers/query` with `context`, `intent`, optional `constraints`, and optional `pagination` |
| Returned offers | Canonical response payload is `request_id` + `offers[]` |
| Offer identity | `offer_id` is the stable inventory Offer; `offer_instance_id` is generated for each served instance |
| Attribution | Preserve `offers[].offer_instance_id` through click, conversion, and settlement flows |
| Categories | 11 canonical public categories in [Category Taxonomy](specs/category-taxonomy.md) |
| Status | `v0.1 Draft`, public beta for the core contract surface |

`v0.1` is implementation-ready for the core Query API, Offer object, schema validation, examples, and postback contracts. Companion drafts such as Agent Identity and Compliance are still evolving.

## Start Here

| Goal | Read | Use next |
|------|------|----------|
| Query offers from an agent or app | [Query API](specs/query-api.md) | [Examples](https://github.com/agentoffernetwork/examples), [Schema](https://github.com/agentoffernetwork/schema) |
| Understand returned `offers[]` | [Offer Schema](specs/offer-schema.md) | [Offer examples](https://github.com/agentoffernetwork/examples) |
| Choose categories | [Category Taxonomy](specs/category-taxonomy.md) | [Offer category attributes](specs/offer-schema.md#category-attributes) |
| Track clicks and conversions | [Events](specs/events.md) | [Postback](specs/postback.md) |
| Check field lifecycle | [Contract Governance](specs/contract-governance.md) | [RFCs](https://github.com/agentoffernetwork/rfcs) |
| Propose a protocol change | [RFCs](https://github.com/agentoffernetwork/rfcs) | [CONTRIBUTING.md](CONTRIBUTING.md) |

Most integrations start with [`POST /v1/offers/query`](specs/query-api.md): copy the minimal request, send `context` + `intent`, read `request_id` + `offers[]`, and preserve `offers[].offer_instance_id` for attribution.

## Developer Paths

| If you want to... | Start here | Then use |
|-------------------|------------|----------|
| Understand the protocol and public positioning | [agentoffernetwork.org](https://agentoffernetwork.org) | This repository for the source specs |
| Make your first request or use mock SDK mode | [Docs Quick Start](https://docs.agentoffernetwork.com/quickstart) | [API Reference](https://docs.agentoffernetwork.com/api), [SDK Reference](https://docs.agentoffernetwork.com/sdk) |
| Integrate an AI app with offers | [Query API](specs/query-api.md) | [Schema](https://github.com/agentoffernetwork/schema), [Examples](https://github.com/agentoffernetwork/examples) |
| Expose partner offers to AON | [OfferProvider API](specs/offer-provider-api.md) | [Partner Integration Guide](https://docs.agentoffernetwork.com/guides/partner-integration) |
| Propose a protocol change | [RFCs](https://github.com/agentoffernetwork/rfcs) | [Contract Governance](specs/contract-governance.md) |

GitHub is the source for protocol semantics, schemas, examples, and RFCs. The docs site provides guided onboarding, field-level API tables, SDK walkthroughs, and runtime platform notes.

## Implementation-Ready Specs

| Specification | Purpose |
|---------------|---------|
| [Query API](specs/query-api.md) | Offer discovery API for agents and SDKs |
| [Offer Schema](specs/offer-schema.md) | Canonical offer object with category, action, entity, bid, and attribution fields |
| [Category Taxonomy](specs/category-taxonomy.md) | Current public category registry and alias boundary |
| [Events](specs/events.md) | Click and conversion event definitions |
| [OfferProvider API](specs/offer-provider-api.md) | Partner-facing offer supply API |
| [Postback](specs/postback.md) | AON-to-Agent and Partner-to-AON callback rules |
| [Contract Governance](specs/contract-governance.md) | Field lifecycle, source references, and stale-field handling |

These specs are published as **v0.1 Draft**. The implementation-ready core is intended for real integration and feedback while the ecosystem matures.

## Companion Drafts

| Specification | Current role |
|---------------|--------------|
| [Agent Identity](specs/agent-identity.md) | Future registration and attribution identity work |
| [Compliance Guide](specs/compliance-guide.md) | Future machine-readable compliance extensions |

## Repository Map

| Repository | Role |
|------------|------|
| [`agentoffernetwork/protocol`](https://github.com/agentoffernetwork/protocol) | Human-readable protocol source |
| [`agentoffernetwork/schema`](https://github.com/agentoffernetwork/schema) | JSON Schema, TypeScript types, and validation guidance |
| [`agentoffernetwork/examples`](https://github.com/agentoffernetwork/examples) | Canonical request/response payloads |
| [`agentoffernetwork/rfcs`](https://github.com/agentoffernetwork/rfcs) | Required path for semantic contract changes |

Markdown explains the contract. Schema and examples make it verifiable. RFCs govern semantic change.

## Current Status

- **Version:** `v0.1`
- **Status:** `Draft`
- **Release posture:** `Public beta for the core contract surface`
- **Stability:** Fields marked REQUIRED are unlikely to change. RECOMMENDED and OPTIONAL fields may evolve.
- **Scope note:** Agent Identity and Compliance remain companion drafts and are not yet reflected as standalone schema artifacts in the v0.1 machine-readable package.

## Contributing

- **Editorial fixes**: open a PR directly.
- **Questions and bugs**: open an issue.
- **Breaking or semantic changes**: open an [RFC](https://github.com/agentoffernetwork/rfcs) first.

See [CONTRIBUTING.md](CONTRIBUTING.md) for detailed routing guidelines.

## Community

- **Website:** [https://agentoffernetwork.org](https://agentoffernetwork.org)
- **Email:** [info@agentoffernetwork.com](mailto:info@agentoffernetwork.com)
- **Security:** [SECURITY.md](SECURITY.md)

## License

This specification is licensed under [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/).
