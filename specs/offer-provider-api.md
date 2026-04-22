# OfferProvider API v0.1

**Version**: 0.1
**Status**: Draft
**Last Updated**: 2026-04-15

## 1. Overview

This document defines the **Partner-side** HTTP contract of the AgentOffer
Protocol — the supply-side interface that the AgentOffer Network (AON)
calls to pull offers from Partners in real time. It is the counterpart of
[`query-api.md`](./query-api.md), which defines the agent-facing interface.

Two kinds of entities implement this contract:

- **Direct Partners** — Partners that own their catalog and serve it
  directly over this API.
- **Bridge Partners** — adapters that translate third-party networks
  (Impact, CJ, etc.) into this API. From AON's perspective a Bridge is an
  ordinary Partner registered with its own credentials; internally a
  Bridge forwards AON queries to upstream networks and maps their
  responses back to AON's envelope.

Offer content returned over this API MUST conform to the Offer Schema
defined in [`offer-schema.md`](./offer-schema.md) and
[`json-schema/offer-schema-v0.1.json`](../../schema/json-schema/offer-schema-v0.1.json).

### 1.1 Conformance Keywords

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHOULD",
"RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

### 1.2 Partner Conformance Levels

Partners MUST declare which conformance level they satisfy. AON validates
Level 1 during onboarding; Level 2 and Level 3 are advisory and feed into
routing weight and deep-integration features.

| Level | Label | Obligation | Summary |
|-------|-------|-----------|---------|
| **Level 1** | Baseline compliance | MUST pass to integrate | Endpoint + signed requests + error envelope + test-mode handling + REQUIRED offer fields |
| **Level 2** | Recommended | SHOULD | Full `filter` support, pagination metadata, test-mode isolation, p95 latency under 2 s |
| **Level 3** | Enhanced | MAY | `/health`, deep pagination, `request_id` echo, multimodal intent handling |

The normative requirement lists for each level are in §12.

## 2. Conformance

A Partner implementation is **protocol-compliant** when it satisfies all
Level 1 requirements below:

- **L1-1** — Exposes `POST {base_url}/v1/offers/query` accepting
  `application/json` bodies shaped per §6.
- **L1-2** — Verifies every incoming request per §4 (HMAC signature,
  timestamp window) before doing any work.
- **L1-3** — Validates `X-AON-Timestamp` against the mandatory ±5-minute
  skew window **independently of** signature verification.
- **L1-4** — Returns responses shaped per §7 on success and per §9 on
  error. All error payloads MUST be valid against the
  [`offer-provider-response-v0.1.json`](../../schema/json-schema/offer-provider-response-v0.1.json)
  ErrorEnvelope definition.
- **L1-5** — Returned offers MUST satisfy every REQUIRED field in
  [`offer-schema-v0.1.json`](../../schema/json-schema/offer-schema-v0.1.json).
- **L1-6** — Honors `X-AON-Test: true` by suppressing tracking, billing,
  fulfillment and conversion side-effects (§5).

Partners SHOULD additionally implement nonce anti-replay (§4.4); this is
RECOMMENDED rather than REQUIRED — see §4.4 for the rationale and the
acceptable degraded mode.

AON validates L1-1 through L1-6 during onboarding using the onboarding
compliance test suite described in §4.5.

## 3. Endpoint

### 3.1 Base URL

Each Partner registers its own **Base URL** with AON (see the Partner
onboarding guide). AON concatenates standard protocol paths onto the
Partner's Base URL; Partners therefore retain full control of domain,
path prefix, and deployment topology.

```
{base_url}  =  e.g.  https://api.partner.example.com/aon
```

### 3.2 Endpoints

| Endpoint | Method | Obligation | Purpose |
|---------|:------:|:----------:|---------|
| `{base_url}/v1/offers/query` | `POST` | MUST | Primary offer discovery endpoint (§6 request, §7 response) |
| `{base_url}/v1/health` | `GET` | MAY | Liveness probe. When implemented, returns HTTP 200 with body `{"code":"SUCCESS","message":"","data":{"status":"ok"},"extra":{}}` |

`Content-Type: application/json` is REQUIRED for the query endpoint on
both request and response.

### 3.3 Versioning

Versioning is carried in the URL path as `/v1/`. Partners embed `/v1/`
after their Base URL (see §3.2). Backward-incompatible changes (including
renaming REQUIRED fields, removing enum values, or tightening validation)
MUST be introduced under a new `/v{N+1}/` path; the old path MUST remain
functional for the deprecation window AON communicates.

## 4. Authentication

AON authenticates itself to Partner via **HMAC-SHA256** over a canonical
signing string. Each Partner is issued an `appkey` and a matching `secret`
at onboarding; AON signs every outbound request; Partner verifies.

### 4.1 Required Headers

| Header | Value | Obligation |
|--------|-------|:----------:|
| `X-AON-Key` | Partner-issued `appkey` identifying AON to the Partner | MUST |
| `X-AON-Timestamp` | Unix epoch seconds (integer, ASCII decimal) | MUST |
| `X-AON-Nonce` | Unique-per-request random string; UUIDv4 RECOMMENDED | MUST |
| `X-AON-Signature` | Lowercase hex-encoded HMAC-SHA256 (see §4.2) | MUST |
| `X-AON-Test` | `true` when request is a test; see §5 | MAY |

### 4.2 Signing String

```
METHOD        + "\n"
PATH          + "\n"
CANONICAL_BODY + "\n"
TIMESTAMP     + "\n"
NONCE
```

- `METHOD` — uppercase ASCII (e.g. `POST`).
- `PATH` — request path only, excluding host and query (e.g.
  `/v1/offers/query`).
- `CANONICAL_BODY` —
  - For `POST` with `application/json`: **the exact UTF-8 body bytes
    AON transmitted on the wire**. Partner MUST use the received raw
    body; it MUST NOT re-serialize, re-order keys, or normalize
    whitespace before hashing. This mirrors the rule already in use by
    the `request_signing` DSL in AON's runtime (see §10).
  - For `GET`: the RFC 3986-sorted, percent-encoded query string with
    no leading `?`.
  - For `POST` with `application/x-www-form-urlencoded`: the sorted
    form body.
- `TIMESTAMP` — identical ASCII decimal string sent in `X-AON-Timestamp`.
- `NONCE` — identical string sent in `X-AON-Nonce`.

No trailing newline. Components are joined by a single LF (`U+000A`).

Compute the signature as:

```
signature_bytes = HMAC_SHA256(secret, signing_string)
X-AON-Signature = hex_lowercase(signature_bytes)
```

Partner MUST compare signatures in **constant time** to prevent timing
side-channels.

### 4.3 Timestamp Skew (MUST)

Partner MUST reject the request when
`|server_now − X-AON-Timestamp| > 300` seconds, returning `401
UNAUTHORIZED` with message `"timestamp outside allowed skew"`. This check
MUST happen before signature verification so that tampered timestamps do
not waste HMAC compute.

### 4.4 Nonce Anti-Replay (SHOULD)

Partner SHOULD maintain a short-TTL set of `(appkey, nonce)` pairs for
the last 5 minutes and reject duplicates with `401 UNAUTHORIZED` and
message `"nonce already used"`. Recommended implementations:

- A Redis `SET` with `PX 300000` `NX` and a `partner:aon:nonce:{nonce}`
  key.
- An in-process TTL map (`LinkedHashMap` + cleanup thread or a
  `Caffeine` cache) for single-instance deployments.

Partners operating in strictly stateless environments (e.g. edge
functions without shared storage) MAY skip nonce enforcement. In that
case they MUST still enforce §4.3 timestamp skew, and they accept a
slightly weaker anti-replay posture. This is why nonce enforcement is
RECOMMENDED rather than REQUIRED.

### 4.5 Onboarding Compliance Tests

During onboarding AON issues the following four test requests against
the Partner endpoint. Partner MUST behave as described for L1
certification:

| # | Scenario | Inputs | Partner expected response |
|---|----------|--------|---------------------------|
| 1 | Valid signature | Fresh timestamp, unique nonce, correct signature | `200 SUCCESS` with a valid success envelope |
| 2 | Tampered signature | Fresh timestamp, unique nonce, random 64-hex signature | `401 UNAUTHORIZED`, `message` describing signature mismatch |
| 3 | Expired timestamp | Timestamp > 5 minutes in the past, correct signature | `401 UNAUTHORIZED`, `message` describing timestamp skew |
| 4 | Replayed nonce | Replay Case 1's nonce within 5 minutes with a fresh timestamp and fresh signature | `401 UNAUTHORIZED`, `message` describing nonce replay — Partners that declared stateless operation (§4.4) return `200` here, and AON records the L1-degraded posture |

Reproducible test vectors (secret, timestamp, nonce, expected signature
hex) for all four cases are published at
[`examples/http/offer-provider/hmac-signing-cases.md`](../../examples/http/offer-provider/hmac-signing-cases.md).

## 5. Test Mode

AON signals a test request in two ways, which MUST carry identical
semantics:

- **Authoritative**: HTTP header `X-AON-Test: true`.
- **Body parity**: `test_mode: true` inside the JSON body (see §6).

When either signal is set to `true`, Partner MUST:

- Return shape-compatible responses that can be used for onboarding
  validation.
- Suppress all tracking, billing, fulfillment, inventory decrement, and
  conversion side-effects.
- Prefer the header value when header and body disagree.

Partner SHOULD clearly tag its own logs to distinguish test from
production traffic. AON MAY choose to mark a Partner degraded if
repeated test requests cause production side-effects.

## 6. Request Body

The request body reuses the canonical `OfferQueryRequest` shape defined
in [`offer-query-schema-v0.1.json`](../../schema/json-schema/offer-query-schema-v0.1.json);
the OfferProvider-specific JSON Schema
([`offer-provider-request-v0.1.json`](../../schema/json-schema/offer-provider-request-v0.1.json))
matches that shape 1-to-1 so that the same request can be validated on
either side of the wire.

The requirement levels follow the same three-tier system used in
`query-api.md`:

| Level | Meaning |
|-------|---------|
| **REQUIRED** | Field MUST be present with a valid, non-empty value. |
| **RECOMMENDED** | Field SHOULD be present and MUST follow the standard structure when present; value MAY be empty. |
| **OPTIONAL** | Field MAY be omitted entirely. |

### 6.1 Top-Level Shape

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `request_id` | string (uuid) | OPTIONAL | UUIDv7 recommended. AON generates it at dispatch; Partners MAY echo it in logs. See also `X-AON-Request-Id` in §12. |
| `timestamp` | string (RFC 3339) | OPTIONAL | AON dispatch time. Distinct from `X-AON-Timestamp`, which is Unix epoch and drives signing. |
| `test_mode` | boolean | OPTIONAL | Mirrors `X-AON-Test`. Header wins on disagreement. |
| `context` | object | REQUIRED | Platform, session, and user context (§6.2). |
| `intent` | object | REQUIRED | Multimodal intent description (§6.3). |
| `filter` | object | OPTIONAL | Hard structured constraints (§6.4). |
| `pagination` | object | RECOMMENDED | Paging control (§6.5). |

### 6.2 `context`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `context.platform.name` | string | RECOMMENDED | e.g. `ChatGPT`, `Claude`, `CustomAgent`. |
| `context.platform.version` | string | RECOMMENDED | e.g. `gpt-4o`. |
| `context.platform.channel` | string | RECOMMENDED | e.g. `plugin`, `sdk`, `api`. |
| `context.session_id` | string | RECOMMENDED | Groups related queries. |
| `context.conversation_id` | string \| number | OPTIONAL | Thread identifier within a session. |
| `context.user_profile.viewer_id` | string | RECOMMENDED | Pseudonymous viewer id. |
| `context.user_profile.language` | string | RECOMMENDED | ISO 639-1. |
| `context.user_profile.interests[]` | array<string> | RECOMMENDED | MAY be empty. |
| `context.user_profile.device_info.device_type` | string | RECOMMENDED | `desktop`/`mobile`/`tablet`. |
| `context.user_profile.device_info.os` | string | RECOMMENDED | `macOS`/`iOS`/`Android`/`Windows`/… |
| `context.user_profile.device_info.os_version` | string | RECOMMENDED | OS version string. |

### 6.3 `intent`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `intent.content[]` | array<object> | REQUIRED | At least one item. Each item has a `type`. |
| `intent.content[].type` | string | REQUIRED | `input_text` or `input_image`. |
| `intent.content[].text` | string | REQUIRED when `type=input_text` | The user utterance. |
| `intent.content[].image_url` | string (uri) | REQUIRED when `type=input_image` | URL of an image for visual search. |

### 6.4 `filter`

All fields are OPTIONAL; when provided they act as hard constraints
applied **before** any semantic ranking Partner performs.

| Field | Type | Description |
|-------|------|-------------|
| `filter.category_types[]` | array<enum> | From `offer_info.category.type`: `software_saas`, `travel_hospitality`, `education`, `financial_service`, `electronics`, `entertainment`. OR-logic within the array. |
| `filter.bid_models[]` | array<enum> | From `bid.model`: `cps`, `cpa`, `cpl`, `cpi`, `hybrid`. OR-logic. |
| `filter.status[]` | array<enum> | From offer lifecycle: `active`, `paused`, `pending`, `rejected`, `expired`. OR-logic. |
| `filter.availability[]` | array<enum> | From `commercial.availability`: `available`, `limited`, `out_of_stock`, `coming_soon`. OR-logic. |
| `filter.min_bid_amount` | string | Decimal string; requires `currency`. |
| `filter.max_price_amount` | string | Decimal string; requires `currency`. |
| `filter.currency` | string | ISO 4217; applies to both `min_bid_amount` and `max_price_amount`. |
| `filter.brand` | string | Case-insensitive substring match against `entity.name`. |
| `filter.country` | string | ISO 3166-1 alpha-2. |
| `filter.tags[]` | array<string> | AND logic across the array. |

Partner SHOULD accept unknown enum values gracefully (return empty results
rather than errors) so that new enum values introduced in a future
revision are non-breaking.

### 6.5 `pagination`

| Field | Type | Level | Default | Limit |
|-------|------|-------|--------:|------:|
| `pagination.limit` | integer | RECOMMENDED | `20` | `100` |
| `pagination.offset` | integer | RECOMMENDED | `0` | — |

## 7. Response Envelope

On success, Partner MUST return `200 OK` with `Content-Type:
application/json` and a body matching the **SuccessEnvelope** in
[`offer-provider-response-v0.1.json`](../../schema/json-schema/offer-provider-response-v0.1.json):

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `trace_id` | string | OPTIONAL | AON-generated Query Trace ID (UUIDv7 recommended). AON injects it at request time and backfills its own value downstream; Partners MAY omit it, MAY return the same value, or MAY return their own correlation id — AON's value always wins. |
| `offers` | array<Offer> | REQUIRED | List of offers conforming to `offer-schema-v0.1.json`. MAY be empty. |
| `has_more` | boolean | OPTIONAL | `true` when the query has further pages. |
| `total` | integer ≥ 0 | OPTIONAL | Total number of matching offers when known. |

### 7.1 Offer Schema Reference

Each element of `offers[]` is an **Offer object** as defined in:

- Human spec: [`protocol/specs/offer-schema.md`](./offer-schema.md)
- JSON Schema: [`schema/json-schema/offer-schema-v0.1.json`](../../schema/json-schema/offer-schema-v0.1.json)
- TypeScript types: [`schema/types/offer.types.ts`](../../schema/types/offer.types.ts)

The OfferProvider response schema does **not** restate Offer fields; it
treats each element as an opaque `object`. AON validates offer structure
against `offer-schema-v0.1.json` downstream.

## 8. Offer Schema Reference

See §7.1 for the authoritative pointers. Partners MUST at minimum populate
every REQUIRED field listed in `offer-schema.md` §"Top-Level Shape":
`uuid`, `version`, `offer_info` (with nested REQUIREDs), `entity`,
`action`, and `bid`. Partners SHOULD populate RECOMMENDED fields such as
`material`, `category.attributes`, and `category.commercial` when the
data is available; routing quality depends on completeness.

## 9. Error Codes

Error responses MUST follow the AON `ApiResponse` contract (consistent
with the rest of AON's HTTP surface):

```json
{
  "code": "BAD_REQUEST",
  "message": "intent.content must contain at least one item",
  "data": {},
  "extra": {}
}
```

| HTTP | `code` | When it happens |
|-----:|--------|-----------------|
| 200 | `SUCCESS` | Normal success envelope (§7). |
| 400 | `BAD_REQUEST` | Malformed body or missing REQUIRED fields. |
| 401 | `UNAUTHORIZED` | Missing auth header, bad signature, expired timestamp, or replayed nonce. |
| 403 | `FORBIDDEN` | Valid `appkey` but suspended / not permitted. |
| 429 | `RATE_LIMITED` | Frequency cap exceeded. Response SHOULD include a `Retry-After` header (integer seconds). |
| 500 | `INTERNAL_ERROR` | Unexpected server-side failure. |

Additional codes MAY be introduced; AON will treat unknown codes as
`INTERNAL_ERROR`-equivalent and may degrade routing weight.

> **Note on divergence from `query-api.md`.** `query-api.md` documents
> the agent-facing interface, where error payloads use `{error:{code,
> message}}`. The OfferProvider API deliberately adopts the project-wide
> `ApiResponse` envelope here because Partner responses are consumed by
> AON's internal offer pipeline, not by external agents.

## 10. Pagination

OfferProvider API uses **offset-based** pagination, aligned with
`query-api.md v0.1`:

- `pagination.limit` bounds page size (default 20, max 100).
- `pagination.offset` skips entries from the start of the result set.
- Partners SHOULD return accurate `has_more` whenever they can determine
  it cheaply (e.g. by fetching `limit + 1` and truncating).
- Partners SHOULD return `total` when inexpensive to compute; MAY omit
  it for large catalogs.
- Deep pagination (`offset > 500`) is a Level 3 MAY-level capability.
  Partners MAY return `BAD_REQUEST` with message `"deep pagination not
  supported"` when they do not support it.

Cursor-based pagination is deliberately not adopted in v0.1 to match
`query-api.md`; a future revision MAY add an opaque cursor alongside
offset.

## 11. Versioning

- The protocol version is surfaced as the URL segment `/v1/` — Partners
  own the full URL, so they embed this segment when constructing their
  Base URL+path.
- Every schema file carries a `$id` suffixed `/v0.1` (the document
  version) so that clients pin both wire version and semantic version.
- Backward-compatible additions (new OPTIONAL fields, new enum values,
  new RECOMMENDED headers) MAY land within `/v1/` in a minor semantic
  version (e.g. `v0.2`); clients are expected to ignore unknown fields
  and unknown enum values.
- Backward-incompatible changes require `/v2/` and a deprecation plan.

## 12. Design Decisions

The 9 decisions below were PMO-locked before Dev Stage; additional
entries document design choices surfaced during technical review.

1. **POST + JSON body.** Moving to POST matches `query-api.md` and lets
   the request carry structured multimodal intent, filters, and user
   profile — impractical to encode in a URL.
2. **Envelope `{ trace_id, offers, has_more, total }`.** `trace_id` is
   AON-generated and AON-authoritative; Partner responses are allowed to
   omit it so Partners do not accidentally become a source of truth for
   a field that belongs to the AON network.
3. **Reuse `OfferQueryRequest` verbatim.** The supply-side request body
   is the exact same shape that agents send. This keeps a single
   authoritative request schema and makes Bridge Partner translation
   trivial when the underlying network already has similar structure.
4. **`ApiResponse` error envelope.** Project-wide consistency with
   `CLAUDE.md` §"API 响应契约" outweighs symmetry with `query-api.md`'s
   agent-facing `{error:{…}}` shape. AON infra (middleware, logging,
   tracing) treats Partner errors identically to any other AON service.
5. **Adapter DSL covers both standard and non-standard Partners.** The
   DSL itself lives outside this document — see
   [`services/offer-mgmt/docs/architecture/PARTNER_CREDENTIALS_DSL.md`](../../../../services/offer-mgmt/docs/architecture/PARTNER_CREDENTIALS_DSL.md).
   Standard Partners leave the adapter unconfigured and receive the raw
   `OfferQueryRequest` / return the raw envelope. Non-standard Partners
   declare a mapping and the runtime reshapes requests/responses. The
   protocol spec intentionally does not embed the DSL so that the
   adapter implementation can evolve without a spec revision.
6. **`test_mode`: header authoritative, body retained.** `X-AON-Test`
   beats the body field on disagreement. The body field remains in the
   shape for parity with `OfferQueryRequest`, and because older Partners
   may already parse it.
7. **`/health` optional.** Mandating `/health` burdens small Partners
   with a second endpoint for little operational upside once routing
   already observes latency and error rate of `/v1/offers/query`
   itself. AON does active probes only when a Partner opts in.
8. **Offset/limit pagination.** Aligned with `query-api.md` and easy for
   Partners with SQL-style catalogs. A future cursor extension is
   compatible.
9. **`/v1/` URL path versioning.** Partners own the Base URL, so
   version-as-path is simpler than header-based versioning for Partner
   implementers. It also lines up with `query-api.md`.
10. **Nonce RECOMMENDED, not REQUIRED.** Very small Partners lack shared
    nonce storage; forcing nonce enforcement raised the implementation
    bar without meaningful security gain beyond the 5-minute timestamp
    window. See §4.4.
11. **Bridge Partners are ordinary Partners.** A Bridge registers in
    AON like any other Partner, signs with its own secret, and is
    responsible for translating upstream network APIs (Impact, CJ,
    AdMitad, …) into this protocol. AON does not prescribe the
    translation; Bridges decide their own internal shape.
12. **`X-AON-Request-Id` header is a MAY.** The HTTP-header mirror of
    the body `request_id` lets Partners extract a correlation id
    without parsing the JSON body (useful in front proxies). Values
    MUST be identical; Partner MAY rely on either.
13. **Apache 2.0 license.** This document inherits the license of the
    Protocol repository (see the `LICENSE` file at the repo root).
14. **Canonical JSON for signing**: The `SORTED_QUERY_STRING_OR_BODY` term in the signing formula refers to *the exact bytes sent on the wire*, not a re-serialized canonical form computed by the verifier. For `POST application/json`, AON's reference implementation serializes the request body with recursively sorted object keys (lexicographic) before sending and signing, which means the bytes the Partner receives are already canonical — Partners MUST sign the raw received body, not a re-serialized form. For `POST form` / `GET`, both sides independently sort key-value pairs ascending before building the signing string. This convention is what `services/offer-mgmt` implements today and what `PARTNER_CREDENTIALS_DSL.md` describes.

## 13. Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-04-15 | Initial draft. Defines endpoint, HMAC-SHA256 auth with ±5-minute skew and SHOULD-level nonce anti-replay, `OfferQueryRequest` reuse, `{trace_id, offers, has_more, total}` success envelope, `ApiResponse` error envelope, offset/limit pagination, Level 1/2/3 conformance, onboarding compliance test matrix, `X-AON-Request-Id` as MAY, and adapter DSL cross-reference. |
