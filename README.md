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
| Agent-facing discovery | `POST /v1/offers/query` with `context`, `intent`, optional `placement_id`, optional `constraints`, and optional `pagination` |
| Returned offers | Canonical response payload is `request_id` + `offers[]` |
| Offer identity | `offer_id` is the stable inventory Offer; `offer_instance_id` is generated for each served instance |
| Attribution | Preserve `offers[].offer_instance_id` through click, conversion, and settlement flows |
| Categories | AON Taxonomy v1 ids in [Category Taxonomy](specs/category-taxonomy.md); current registry includes 24 Level 1 categories and deeper child ids |
| Location targeting | `targeting[].geo.include/exclude` use AON Location Registry v1 `location_id` values for COUNTRY, REGION, and CITY; Location Search/Resolve API defines lookup for text, ISO 3166-2, CLDR, Cloudflare, and Google Cloud signals; legacy country strings remain migration-compatible |
| Age eligibility | `targeting[].eligibility.min_age` pairs with Query `context.user_profile.verified_age_over[]`; do not send DOB or exact age |
| Status | `v0.1 Draft`, public beta for the core contract surface |

`v0.1` is implementation-ready for the core Query API, Offer object, schema validation, examples, and postback contracts. Companion drafts such as Agent Identity and Compliance are still evolving.

## Start Here

| Goal | Read | Use next |
|------|------|----------|
| Query offers from an agent or app | [Query API](specs/query-api.md) | [Examples](https://github.com/agentoffernetwork/examples), [Schema](https://github.com/agentoffernetwork/schema) |
| Understand returned `offers[]` | [Offer Schema](specs/offer-schema.md) | [Offer examples](https://github.com/agentoffernetwork/examples) |
| Choose categories | [Category Taxonomy](specs/category-taxonomy.md) | [Offer Schema category field](specs/offer-schema.md#offer_infocategory) |
| Apply location or age eligibility | [Offer Schema targeting](specs/offer-schema.md#targeting-optional) + [Query API context](specs/query-api.md#device-os-location-and-age-context) | [Location Search API](specs/location-search-api.md), [Location registry](https://github.com/agentoffernetwork/schema/blob/main/locations/aon-location-registry-v1.json) |
| Track clicks and conversions | [Events](specs/events.md) | [Postback](specs/postback.md) |
| Check field lifecycle | [Contract Governance](specs/contract-governance.md) | [RFCs](https://github.com/agentoffernetwork/rfcs) |
| Propose a protocol change | [RFCs](https://github.com/agentoffernetwork/rfcs) | [CONTRIBUTING.md](CONTRIBUTING.md) |

Most integrations start with [`POST /v1/offers/query`](specs/query-api.md): copy the minimal request, send `context` + `intent`, optionally include a top-level `placement_id` when your platform has a placement context, read `request_id` + `offers[]`, and preserve `offers[].offer_instance_id` for attribution.

## Developer Paths

| If you want to... | Start here | Then use |
|-------------------|------------|----------|
| Understand the protocol and public positioning | [agentoffernetwork.org](https://agentoffernetwork.org) | This repository for the source specs |
| Make your first request or use mock SDK mode | [Docs Quick Start](https://docs.aon.pro/quickstart) | [API Reference](https://docs.aon.pro/api), [SDK Reference](https://docs.aon.pro/sdk) |
| Integrate an AI app with offers | [Query API](specs/query-api.md) | [Schema](https://github.com/agentoffernetwork/schema), [Examples](https://github.com/agentoffernetwork/examples) |
| Expose partner offers to AON | [OfferProvider API](specs/offer-provider-api.md) | [Partner Integration Guide](https://docs.aon.pro/guides/partner-integration) |
| Propose a protocol change | [RFCs](https://github.com/agentoffernetwork/rfcs) | [Contract Governance](specs/contract-governance.md) |

GitHub is the source for protocol semantics, schemas, examples, and RFCs. The docs site provides guided onboarding, field-level API tables, SDK walkthroughs, and runtime platform notes.

## Implementation-Ready Specs

| Specification | Purpose |
|---------------|---------|
| [Query API](specs/query-api.md) | Offer discovery API for agents and SDKs |
| [Offer Schema](specs/offer-schema.md) | Canonical offer object with category, action, entity, bid, and attribution fields |
| [Location Search API](specs/location-search-api.md) | Lookup contract and migration guidance for AON Location Registry ids |
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
| [`agentoffernetwork/schema`](https://github.com/agentoffernetwork/schema) | JSON Schema, TypeScript types, AON Taxonomy data, and AON Location Registry data |
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
- **Email:** [info@aon.pro](mailto:info@aon.pro)
- **Security:** [SECURITY.md](SECURITY.md)

## License

This specification is licensed under [Creative Commons Attribution 4.0 International](https://creativecommons.org/licenses/by/4.0/).
