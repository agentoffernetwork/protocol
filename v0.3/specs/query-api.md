# Query API

> **Current stable contract**
>
> New integrations use wire version `0.3`. Each API, MCP, Provider, and Host
> deployment publishes its own endpoint, access, and conformance details.

## Version negotiation

Only the exact request header `AON-Protocol-Version: 0.3` selects this
contract. A service that has not deployed v0.3 returns
`unsupported_protocol_version` rather than silently interpreting the request as
v0.2. The response echoes `AON-Protocol-Version: 0.3` when v0.3 is actually
supported. Shared caches must vary on this header.

New integrations should send `AON-Protocol-Version: 0.3`. Requests without a
version header follow the deployment's documented rollout state. Requests with
`0.2` keep the existing v0.2 behavior and must never be silently reinterpreted
as v0.3.

## Request

The request carries the current user intent, not a complete conversation log.
`intent.content` is the user's current text or image input. `intent.provenance`
distinguishes an explicitly expressed intent from a bounded inference. The
optional `context.session` carries only a previous request id and a short list of
recent topics. Recent topics are mapped to offer categories and may boost ranking
when a recalled offer overlaps those categories; they do not change recall.

`intent.origin` is an array of `{ kind, id }` references. Consumers deduplicate
by `kind + id`, retain at most three entries, and do not evict older entries when
the limit is reached. Long-term profiles and raw chat transcripts are not part of
this public request.

`force_offer` defaults to `false`. When true, it permits the runtime to select a
qualified fallback Offer if the exact query has no match. It never bypasses
eligibility, freshness, sensitive-category, or action gates.

`response_options.thinking_mode` defaults to `true`. When false, the response
omits `offers[].match_reason`; when true, `match_reason` may contain a concise,
user-facing explanation and must not expose internal chain-of-thought or private
ranking data.

## Response

Every response has `request_id`, `protocol_version`, `language`, and `offers`.
Offers can be empty. When `offers` is empty, `empty_reason` is required and uses one of:
`frequency_capped`, `below_relevance_threshold`, `scene_suppressed`,
`no_material`, or `consent_missing`. When an Offer is returned, `empty_reason`
is omitted.

`engagement.refinements` helps narrow the current request and carries a short
`label`, an optional `speak` suggestion, and an item-level `query_helper`.
`engagement.followup_topics` helps the user explore a related direction and
carries a `label`, `basis`, confidence score, and item-level `query_helper`.
v0.3 does not define a top-level
`engagement.query_helper`, generic `next_actions`, or `decision_factors`.

`query_helper.request_patch` uses RFC 7386 merge-patch semantics and is limited to
`intent.signals`, `constraints.category_ids`, and
`constraints.excluded_category_ids`. User-provided conditions from the current
turn win over conflicting patch values. The merged request is validated again
before it is sent.

## Offer boundaries

`entity` identifies the merchant, brand, provider, or publisher. Optional
`listing_source` describes where the user-facing listing information was observed
(for example a marketplace or official site). `action` is the executable
destination. The three fields are not interchangeable. `listing_source` is omitted
when it is missing required data, stale, non-UTC, inconsistent with the entity, or
contains provider/affiliate/internal parameters.

`listing_source.logo` is an optional explicit platform/site Logo, not a copy of
`entity.logo`, `action.payload.url`, or `material[]`. It must be an absolute
HTTPS URI of at most 2048 characters; non-ASCII components must be
percent-encoded. Consumers fall back to
`listing_source.name` when it is absent. Query projection removes only an
invalid optional Logo from trusted historical data; if `kind`, `name`, or
`observed_at` is invalid, it omits the entire source block.

## Runtime boundary

REST handler, Provider Adapter, fallback pool, `feedback`, `watches` persistence,
and MCP registration are deployment-owned implementation work. The v0.3
contract is adopted independently of runtime rollout: each deployment must
publish conformance evidence before accepting v0.3 traffic. Deployments that do
not support v0.3 return `unsupported_protocol_version`; they must not silently
reinterpret the request as v0.2. v0.2 remains the explicit compatibility path.
