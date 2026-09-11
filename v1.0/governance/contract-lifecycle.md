# Contract Lifecycle

AgentOffer Protocol separates contract adoption from deployment rollout. This
keeps normative semantics stable while allowing each runtime owner to publish
its own endpoint, access, and availability evidence.

## Current status

| Property | Value |
|---|---|
| Contract | AgentOffer Protocol v1.0 |
| Lifecycle | Adopted |
| Stability | Stable |
| New integrations | Default contract |
| Runtime support | Deployment-owned |
| Version selection | Exact `1.0`; every other value fails closed |

The public governance registry records `offers.query/public-v1.0` as adopted,
stable, and the default for new integrations. Earlier releases remain immutable
provenance and recovery evidence rather than alternate current paths.

## Authority

| Layer | Authority | Responsibility |
|---|---|---|
| Semantics | Protocol specifications | Field meaning, cross-field rules, and version behavior |
| Structure | JSON Schema and semantic validators | Requiredness, closure, formats, branches, and executable constraints |
| Examples and types | Published projections | Copyable payloads and implementation aids that follow the contract |
| Runtime HTTP | Deployment OpenAPI and runtime | Endpoint, authentication, status, rollout, and operational behavior |
| Change decisions | RFC repository and governance registry | Durable semantic decisions and lifecycle status |

Specifications and machine-readable contracts must agree. Examples, SDKs,
websites, and runtime documentation follow them and must not create an
independent wire contract.

## Version selection

New integrations select the current contract with exact
`AON-Protocol-Version: 1.0`. Missing, historical, unknown, range, and approximate
selectors fail closed; a deployment must not silently reinterpret one contract
as another.

## Runtime boundary

Publication means that the contract is available for implementation and
review. It does not by itself enable traffic, grant credentials, approve a
Partner, or prove that a deployment is conformant. Each deployment owner is
responsible for rollout evidence and for documenting whether the current
contract is accepted.

## Changing the contract

Wire-level behavior, field semantics, compatibility, and governance changes
use the [RFC process](https://github.com/agentoffernetwork/rfcs). Editorial
clarity, navigation, and broken-link fixes may use a direct pull request when
they do not change semantics.

The initial v1.0 promotion carried forward the already governed field and
behavior semantics without inventing a retroactive RFC. Later v1.0 semantic
changes follow the live process before contract admission. RFC-0004 is the
accepted and implemented decision for the response-scoped Offer display price;
RFC-0005 is the accepted and implemented decision for Flight airport-local
schedule times and source-provided segment duration.

RFC-0006 is the accepted decision for optional, independent Query
`alternative_offers`. It preserves existing request and main-result behavior,
including `force_offer`, on exact v1.0 and the next unused protected rN.
Unextended old responses remain valid, but old closed readers may reject the
new field and permissive readers may discard it. Consumer read support must
be upgraded and certified before producer emission is enabled; this is not
a universal backward-compatibility guarantee.

Query alternative wrappers are response-owned, not Partner/OfferProvider supply
fields or an expansion of the Generic Offer projection. Their schema,
semantics, types, examples and RFC must enter the existing protected release
admission together. Local source completion, public protocol publication,
documentation deployment and runtime certification are separate evidence
states. Service, SDK and Agent adaptation remains deployment-owned; neither
an accepted RFC nor a synthetic example establishes live availability.
