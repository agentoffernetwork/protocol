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


RFC-0010 is the accepted decision for optional Query constraints
`offer_types` and `listing_source_names`; it does not add `listing_source_kinds`.
Omission preserves existing inference, nonempty arrays override inference in
the same dimension, and explicit empty arrays clear that dimension's AON
filter and suppress inference for that request. These semantics do not control
third-party Provider search algorithms. Exact `1.0` selection is unchanged.
Old closed readers can reject the new fields, so contract adoption alone is
not a capability signal for an endpoint or Provider.

RFC-0011 is accepted and implemented in candidate sources for optional Flight
source airport names, terminals, planned aircraft names and stop-name language.
Its Schema, types, semantics and examples require protected contract-extension
admission and publication before they become a public revision. Strict-reader
deployment, verified source field coverage and product-owner approval of
nonstop retention must precede live producer activation. Existing strict
third-party readers may reject Offers with the new optional keys, so their
upgrade requirement must be stated in release notes.

RFC-0012 is accepted and implemented in candidate sources for the `game`
supply profile and its required positive `downloads` fact. The profile registry,
Schema, types, semantic validator and specifications require protected
admission and publication. Strict readers must support the third profile before
producers emit it; RFC acceptance alone does not establish runtime support.

RFC-0013 is accepted and implemented in candidate sources for optional Flight
purchase display facts: carrier logos and verified operating identity, airport time zones, explicit
airport-changing connections, standardized planned equipment, cabin amenities,
fare-specific conditions and estimated emissions. Its Schema, types,
validator, specification and example changes require protected
contract-extension admission and publication before they become a public
revision. Reader compatibility checks are required before producer emission.
Data acquisition and enrichment are separate; the candidate makes no supplier
coverage or runtime claim.

The `query_constraint_filters` extension binds the canonical Schema, types,
semantic validator, fixtures, specifications and RFC closure to one immutable
candidate commit through existing protected release admission. This change
updates protocol sources and release preparation only; it does not adapt
runtime services, SDKs, MCP, Action, OpenAPI, sites, databases or search indexes.
Publication uses the next unused protected revision when separately authorized;
old release manifests and receipts remain immutable. A deployment must provide
its own support evidence before callers send the new constraints.
