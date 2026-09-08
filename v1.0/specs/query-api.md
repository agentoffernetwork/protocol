# Query API

> **Current stable contract**
>
> The stable v1.0 canonical line uses wire version `1.0`. Each API, MCP,
> Provider, and Host deployment publishes its own endpoint, access, and
> conformance details.

## Version negotiation

Only the exact request header `AON-Protocol-Version: 1.0` selects the stable
v1.0 contract. A conforming response echoes `AON-Protocol-Version: 1.0`, carries
`protocol_version: "1.0"` in its JSON body, and sends
`Vary: AON-Protocol-Version`. Offer payload structure and field constraints are
defined by the v1.0 Offer schemas and semantic validator; payload fields do not
participate in transport version negotiation.

The selector is mandatory. Missing values and every value other than exact
`1.0`, including historical, range, and approximate values, return
`unsupported_protocol_version`. A conforming v1.0 implementation does not
silently fall back or reinterpret a request. A future protocol line must publish
its own version namespace policy before it can be selected.

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

The request does not declare a target display currency and does not define
`response_options.price_currency`. Intent parsing, session preferences, and
target-currency selection are implementation-owned inputs to the response, not
a second public request contract.

## Response

Every response has `request_id`, `protocol_version`, `language`, and `offers`.
Offers can be empty. When `offers` is empty, `empty_reason` is required and uses one of:
`frequency_capped`, `below_relevance_threshold`, `scene_suppressed`,
`no_material`, or `consent_missing`. When an Offer is returned, `empty_reason`
is omitted.

Returned Offers are ordered by descending selection priority for this request.
The ordering is meaningful only within the response and is not stable or
comparable across separate requests. When several empty-result causes apply,
the canonical precedence is `consent_missing`, `scene_suppressed`,
`frequency_capped`, `no_material`, then `below_relevance_threshold`.

The normative meaning of every returned Offer and response-envelope property is
defined in [Offer and Query Response Field Semantics v1.0](offer-field-semantics.md).
The published JSON Schema `description` annotations are the authoritative
field-level contract; this API specification defines cross-field and transport
behavior.

The current `offers[]` response is intentionally the Generic Offer projection.
It may carry the response-owned
`offer_info.commercial.display_price` defined below. It rejects and does not
emit `offer_info.details`,
`offer_info.commercial.price.tax_status`, or
`offer_info.commercial.quote`, even if the source Offer carries a registered
Flight or Hotel Rate supply profile. This is a projection boundary, not a new
selector or Offer version: the only v1.0 selector remains
`AON-Protocol-Version: 1.0`, and the Offer document marker remains `"3.0"`.
A later runtime projection requires separately certified deployment evidence.

### Response display price

Each returned Offer may contain one optional, closed
`offer_info.commercial.display_price` object with exactly string `amount` and
`currency`. It is generated for the current Query response; Partner Offers and
OfferProvider success Offers must not supply it. When present it requires an
original `commercial.price`, uses a different currency, and follows the same
canonical decimal grammar and zero-value class as that original price.

The normative consumer rule is:

```text
effective_display_price =
  display_price exists
    ? { ...price, amount: display_price.amount, currency: display_price.currency }
    : price
```

Only `amount` and `currency` are overlaid. Existing `price.unit` and any
available `price.tax_status` retain only their source-price meaning. The
Generic Query projection does not currently carry `tax_status`; consumers must
not infer it. Only complete absence permits fallback to `price`. If
`display_price` is present but null, partial, malformed, same-currency, or
otherwise invalid, the response violates the contract and must not be silently
rendered using the fallback branch.

`display_price` is a response-scoped presentation value. It is never a
checkout, settlement, or transaction-authoritative price. The original
`commercial.quote.observed_at` and `valid_until` do not establish
display-price or FX freshness. The wire object intentionally provides no FX
source, FX observation time, rounding method, or validity evidence. Consumers
must not infer that a conversion is correct or current.

`engagement.refinements` helps narrow the current request and carries a short
`label`, an optional `speak` suggestion, and an item-level `query_helper`.
`engagement.followup_topics` helps the user explore a related direction and
carries a `label`, `basis`, confidence score, and item-level `query_helper`.
v1.0 does not define a top-level
`engagement.query_helper`, generic `next_actions`, or `decision_factors`.

`query_helper.request_patch` is a constrained non-destructive partial update limited to
`intent.signals`, `constraints.category_ids`, and
`constraints.excluded_category_ids`. Omitted members remain unchanged, objects
merge recursively, arrays replace prior arrays, and `null` is invalid rather
than a removal instruction. User-provided conditions from the current turn win
over conflicting suggested values. The merged request is validated again before
it is sent.

`followup_topics` are ordered by descending `confidence`. Confidence is
comparable only among follow-ups in the same response from the same producer; it
is not a calibrated probability or a cross-request score.

Every Hook identifies one returned Offer with `subject_offer_id` and one prior
response with `baseline_request_id`. The baseline must equal the current
request's `context.session.previous_request_id`. Hooks report observed change
cues only; they do not register a watch or promise future notification.

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
and MCP registration are deployment-owned implementation work. The v1.0
contract is adopted independently of runtime rollout: each deployment must
publish conformance evidence before accepting v1.0 traffic. Deployments that do
not support v1.0 return `unsupported_protocol_version`; canonical publication
alone is not evidence that a deployment accepts v1.0 traffic.
