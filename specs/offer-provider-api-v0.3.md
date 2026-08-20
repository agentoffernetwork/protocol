# OfferProvider API v0.3

> **Canonical v0.3 contract**

Provider request reuses `offer-query/v0.3` and requires `request_id`. Provider
success reuses `offer-query-response/v0.3`; Provider errors use the existing
uppercase `code`, `message`, `data`, `extra` envelope (`BAD_REQUEST`,
`UNAUTHORIZED`, `FORBIDDEN`, `RATE_LIMITED`, `INTERNAL_ERROR`). HMAC, nonce and transport metadata
remain headers, not public body fields.

Provider adapters may use private eligibility, freshness, mapping and supply
lineage data internally, but those fields must not leak into the public response.
The public Offer projection is the same shape as Query, including the semantic
separation of `entity`, `listing_source` and `action`.
