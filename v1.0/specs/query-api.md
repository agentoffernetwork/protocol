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
Offers can be empty. In the Generic branch (no `flight_search`), when `offers` is empty, `empty_reason` is required and uses one of:
`frequency_capped`, `below_relevance_threshold`, `scene_suppressed`,
`no_material`, or `consent_missing`. When a main Offer is returned, `empty_reason`
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

Without `intent.details`, `offers[]` uses the Generic Offer projection.
It may carry the response-owned
`offer_info.commercial.display_price` defined below. It rejects and does not
emit `offer_info.details`,
`offer_info.commercial.price.tax_status`, or
`offer_info.commercial.quote`, even if the source Offer carries a registered
Flight or Hotel Rate supply profile. This is a projection boundary, not a new
selector or Offer version: the only v1.0 selector remains
`AON-Protocol-Version: 1.0`, and the Offer document marker remains `"3.0"`.
A later runtime projection requires separately certified deployment evidence.

### Optional alternative Offers

In the Generic branch only, after the complete existing main Query path has finished, an empty `offers`
may be accompanied by `alternative_offers`. This optional response field does
not change the request. In particular, a successful `force_offer` fallback
remains in main `offers`; it is not moved into the alternative list. Main
Offers and alternatives are mutually exclusive. `empty_reason` continues to
describe the empty main result, even when alternatives are present.

When present, `alternative_offers` is a non-null array of **1–3** closed items:

| Field | Contract |
| --- | --- |
| `basis` | Required; initially only `regional_popularity`. |
| `selection_reason` | Required request-specific explanation, in response `language`, of why this alternative was selected rather than a direct match. |
| `offer` | Required complete current Generic Query Offer projection, without `match_reason`. |

`selection_reason` contains 1–500 Unicode code points and at least one
non-whitespace character. Whitespace is the ECMAScript `\s` set; length is not
UTF-16 code units or bytes. No trimming or normalization is implied. The reason
must truthfully disclose the current selection basis, not claim a direct match
to the original query, and must not expose private ranking data or internal
chain-of-thought. `thinking_mode=false` does not remove this required disclosure.
Alternative Offers must not contain `match_reason` in either thinking mode.
Static Partner/provider `offer_info.recommendation_reason` may remain on the
Offer, but cannot replace the request-specific `selection_reason`.

Items must be unique by stable `offer.offer_id` (UUID case variants identify
the same Offer), even if their `offer_instance_id` or explanation differs.
Do not merge alternatives into main `offers` or main-result counts. Omit the
whole field when no suitable alternatives exist or the alternative branch
fails; `null` and an empty array are invalid substitutes for omission.

Alternatives are permitted only with `below_relevance_threshold` or
`no_material`. They are forbidden with `consent_missing`, `scene_suppressed`,
or `frequency_capped`; all five meanings and their precedence remain unchanged.
The producer must evaluate the **real internal causes and global gates** before
selecting alternatives. A public reason can collapse several internal causes:
`below_relevance_threshold` alone is not authorization to bypass consent,
scene, frequency, or other eligibility restrictions.

`regional_popularity` requires genuine, current popularity evidence for the
user's same country and qualified inventory. The producer must not infer a
country from response `language`. Unknown country, no applicable popularity
list, or no qualified candidate requires omission. Only natural-language
relevance may be relaxed: explicit included and excluded categories, budget,
other explicit exclusions, eligibility, freshness, sensitive-category, and
action gates still apply. Cross-category alternatives are allowed only when
there is no explicit included-category restriction, and must still respect
excluded categories. A `no_material` main result does not excuse an alternative
from having usable action and presentation resources.

Catalog/placement-scoped queries, Browse pagination exhaustion, and the public
test sandbox must not trigger global-popularity refill. In paired validation,
a request carrying `placement_id` or `test_mode=true` forbids alternatives;
internal routing and Browse state remain producer obligations. Validation
without the request cannot certify entry-point eligibility.

The nested Offer retains the existing dispatch identity, action, attribution,
and display-price contracts; it does not admit supply-only `details`, `quote`,
or `price.tax_status`. Returning an alternative is not proof of actual display
and creates no new charging event. `engagement` is unchanged, and Hooks still
reference **main `offers` only**, never alternatives.

JSON Schema and semantic validation prove payload constraints, not real
popularity, country, eligibility, internal gates, or the truth/language of an
explanation. Those require deployment evidence. The
[alternative Query example](https://github.com/agentoffernetwork/examples/blob/main/v1.0/http/offer-query-alternative-offers.json)
is synthetic contract data, not runtime evidence.

This extension stays on exact v1.0 through the next unused protected rN; the
Offer document marker remains `"3.0"`. Existing responses without the field
remain valid. **Old closed readers may reject the new field; permissive readers
may discard it.** Consumers must upgrade and certify reading before producers
enable emission. Canonical publication does not certify deployment support;
service, SDK, and Agent adaptation is separate implementation work.
`force_offer` remains supported under its existing rules; any later retirement
requires a separate decision after capability rollout and caller migration.

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

Every Hook identifies one main `offers` item with `subject_offer_id` and one prior
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

## Typed Flight Query

`intent.details` opts into the closed `{profile: "flight", data: {...}}`
request profile. Without it, existing Generic rules apply. No other Query
profile is admitted. A typed request MUST invoke capable sources for this
request; stored examples or unknown-age cached prices do not establish a live
search. Execution uses the existing synchronous response within an implementation-declared
finite deadline. Supplier task polling is an adapter detail and must finish or be
classified as a timeout within that deadline; this extension adds no asynchronous
response or later result updates. One Offer represents one
complete candidate itinerary, including connecting segments, price and action.

| `intent.details.data` field | Contract |
| --- | --- |
| `query_kind` | Required `reference_search` or `traveler_quote`; no inferred default. |
| `legs` | Required nonempty ordered array; each leg has `origin`, `destination`, and `departure_date`. |
| `legs[].origin`, `destination` | Closed `{kind: "airport" or "city", code: "AAA"}`; uppercase three-letter code; identical kind/code endpoints forbidden. Syntax is not registry verification. |
| `legs[].departure_date` | Explicit valid calendar date `YYYY-MM-DD` in the origin's local calendar; never default tomorrow. |
| `travelers` | Forbidden for reference search; required for traveler quote. Unique `adult`, `child`, `infant` groups with positive integer `count`. |
| `travelers[].ages` | Whole nonnegative years on the first leg's local departure date; required for child/infant, optional for adult; length equals count. |
| `travelers[].infant_seat_required` | Required boolean array for infant, same index as ages and same length as count; forbidden for other types. |
| `cabin_class` | Optional `economy`, `premium_economy`, `business`, `first`; every actual segment must match exactly. |
| `max_connections` | Optional nonnegative integer; each leg's `segments.length - 1` must not exceed it. |
| `nonstop_only` | Optional boolean; true requires exactly one segment per leg and explicit `stops: []`. |

Objects are closed. Existing `intent.signals.budget` remains the budget input;
there is no new maxPrice, locale, market, airline filter or currency selector.
Ordered legs express one-way, return and multi-city intent without a new
request `trip_type`. Cross-leg local dates need not be monotonic. Implementers
publish body-size and supplier limits and reject excess rather than truncating
legs or passenger counts. The protocol invents no universal passenger limit,
adult accompaniment ratio or age-category cutoff. A supplier needing birth
dates, unable to honor requested ticket categories under its actual age/seat
rules on any leg, or lacking whole-itinerary pricing MUST report unsupported
capability. Independent one-way prices cannot be added into a guaranteed
complete itinerary quote.

### Flight results and pairing

Every typed success has closed `flight_search` with required `query_kind`
(equal to the request), `status` (`complete` or `partial`), and RFC3339
`fetched_at`. This timestamp is when collection for this search batch finished,
not source quote creation or expiry. `complete` requires at least one capable
source actually queried, with every selected capable source completing.
`partial` requires at least one completed source and at least one failure,
timeout or unusable candidate source. It can contain zero offers. It never
means a complete no-match. All participating sources failing is an error.
Complete coverage describes selected sources, not the whole market. A Provider
reporting partial MUST propagate uncertainty to the enclosing Query as partial;
an HTTP success cannot promote partial upstream collection to complete.

Typed main `offers` MUST use the Flight projection, carry `offer_info.details`,
and explicitly declare `details.data.price_basis`: `reference` for reference
search (no travelers), `itinerary_total` for traveler quote (all requested
travelers and legs). Typed responses MUST omit `empty_reason` and
`alternative_offers`. Only `complete` plus `offers: []` means no match in the
completed selected-source search. Reference prices have unknown traveler
scope; they are not asserted per-person prices or party totals. Typed price
`unit` is absent or `one_time`. Original price, tax state and executable
`book` action must describe the same candidate. Public identity, attribution,
match_reason, and response-owned display_price rules remain applicable.

The conforming validation pipeline first validates request and response with
JSON Schema (for example AJV), then calls the corresponding pure semantic
validator with the complete request and trusted city evidence. Semantic helpers
do not replace structural validation or load an airport directory.
Semantic validation requires a complete paired request, including when the
response contains zero offers. A request with details and a response without
flight_search, or the reverse, is invalid. Existing request_id correlation
rules apply. For every offer, legs must match request order and count; the
first departure and last arrival match endpoints, first departure local date
matches the requested date, and all segment cabins and connection/stop limits
match. Airport endpoints match by code. City endpoints require trusted airport
directory or explicit supplier city-membership evidence; code syntax, prose,
URL parameters and self-reported response text are insufficient.

The local validation API accepts `evidence.airportCityCodes` as a mapping from
airport code to city-code arrays, supplied from independent trusted facts.
This is not a wire field or a new directory service. Missing city evidence
makes a candidate unverifiable, not a match. Travelers match types/counts and
per-type age/seat multisets, preserving requested ages without inventing adult
ages. Unknown stop state cannot satisfy nonstop. `force_offer`, ranking,
category signals and natural language cannot relax these conditions; hooks or
engagement request patches cannot delete or overwrite them.

### Flight errors and execution boundaries

Failures use the existing uppercase `{code, message, data: {}, extra}` envelope,
not the success schema. Portable protocol errors use the dedicated
`offer-query-error-v1.0.json` entry, sharing Provider error-envelope definitions.
This entry and `OfferQueryErrorV10` are not an exhaustive union of hosted
deployment errors. Deployment-specific errors (for example, placement/catalog
errors or transport version negotiation failures) retain their deployment-owned
codes and payloads and MUST be validated against that deployment's error
contract, not rejected or rewritten to fit this portable envelope.
`extra.flight_search_error` is closed `{kind, fields?}`; optional `fields` is a
nonempty array of JSON Pointers to input constraints.

| Outcome | Representation |
| --- | --- |
| Invalid or incomplete input | `BAD_REQUEST` / `invalid_query`; no supplier call. |
| No source can honor constraints or provide required matching facts | `BAD_REQUEST` / `unsupported_capability`; no silent Generic fallback. |
| All participating sources fail, time out or return unusable candidates | `INTERNAL_ERROR` / `upstream_failure`. |
| Some complete and some fail | Success with `status: "partial"`; zero or more compliant offers. |
| All selected capable sources complete with no match | Success with `status: "complete"`, `offers: []`. |

Typed BAD_REQUEST/INTERNAL_ERROR require this discriminator; any supplied kind
must pair with the corresponding code. Authentication, rate limits and policy
failures keep existing codes. Consent/scene/frequency rejection before querying
uses `FORBIDDEN` and an explanatory message, never fabricated complete/partial
or upstream_failure. Do not disclose supplier credentials or user IP. There is
no protocol retry, cached fallback, automatic downgrade or alternative refill.

Supply observation and expiry rules are described in
[Flight price and itinerary facts](offer-field-semantics.md#flight-price-and-itinerary-facts).
A real-time lookup does not lock price, inventory, booking or ticket issuance;
implementers remain responsible for actual calls, correct source evidence and
consistent actions. Validators cannot prove those external facts. Revalidation
may be needed before purchase. The [Ctrip mapping](flight-query-ctrip-mapping.md)
records the specific supplied tool evidence without certifying live support.

### Compatibility and availability

This opt-in source contract retains exact selector `1.0` and Offer marker
`3.0`; it uses a new protected v1.0 rN, not a profile_version or new endpoint.
Old valid Generic and supply instances retain their semantics. Older closed
validators can reject new typed instances. Deployment integration documentation
MUST explicitly declare support for this revision before clients enable it;
`protocol_version: "1.0"` alone is not capability discovery. No declaration
means unsupported. RFC-0007 acceptance and source implementation do not prove
public publication, runtime deployment, SDK/Agent readiness or live supplier
execution; each requires separate evidence.
