# Ctrip Flight Query mapping

This adapter guidance is based on supplied `query_flight` tool documentation
screenshots and two historical JSON sample sets, not an independently verified
live call. The source tool is described as Streamable HTTP MCP. Credentials
and source tracking identifiers are excluded. The generic contract is
[typed Flight Query](query-api.md#typed-flight-query); these supplier details
do not create new AON fields or certify deployed capability.

| Source parameter/fact | Mapping and boundary |
| --- | --- |
| departCityCode/fromCity; arriveCityCode/toCity | Each pair accepts city code or name; prefer verified city identity. Never infer airport membership from name or URL. |
| fromAirport/toAirport | Airport names, supplied with corresponding city; airport match takes precedence. An AON airport code cannot be sent as a name without trusted mapping. |
| departDate | Always send explicit request local date; do not inherit the tool's tomorrow default. |
| allFlights | Defaults false, limited to 10 results; true says all tool results, not whole-market coverage. Complete means selected sources completed, not exhaustive inventory. |
| directFlight | true requests direct flights; false does not particularly select connections. Recheck segment count and explicit stop facts for nonstop_only. |
| airSpace | Y,S prefers economy then premium economy; C,F prefers business then first; C and F are individual cabin options. A fallback cannot satisfy a strict different cabin request. Check every actual segment. |
| airLine | Airline code; marketing-versus-operating match semantics unconfirmed. No universal AON airline filter is introduced. |
| maxPrice | No independent currency or traveler price-basis input was documented. Do not map budget without verified comparable currency and scope. |
| sortType | 1 price ascending, 2 departure ascending, 3 departure descending, 4 popularity; adapter detail only. |
| locale | Default zh-CN; zh-CN uses Ctrip links, other locales Trip links; documented zh-HK corresponds to HKD. Do not extrapolate other locale-to-currency mappings or infer channel/currency from AON response language. |
| Traveler/return/multi-city inputs | Not documented in the supplied tool; do not silently discard them or add one-way prices into a complete itinerary guarantee. Until independently supported, reject traveler_quote or unsupported multi-leg capability. |

Map source ordered route segments to one Offer itinerary, preserving airport
codes, local dtime/atime (space-to-T normalization only), marketing carrier,
flight number and verified cabin. Preserve segment source duration. Empty
airport/carrier labels do not justify invented names; required facts still
need evidence. All 14 historical routeType values were 2, so routeType cannot
be used to infer connection count or AON trip_type.

The supplied connecting example had leg duration 2765 minutes and segment
durations 210, 115 and 435 (sum 760). Preserve the leg total including waits;
do not replace it with the sum or subtract clocks across time zones. Source
`stops: [{"name":"嘉峪关","duration":60}]` maps to
`stops: [{"name":"嘉峪关","duration_minutes":60}]`. Do not invent an airport
code for a source city label. Missing stops are unknown, never an empty array;
a name-only stop still defeats nonstop. Zero/unknown duration is omitted while
known location facts are retained. Connections remain separate from stops.

Source pl[].price and currency can inform original price. isContainsTax must
be interpreted only to the coverage it actually establishes; otherwise use
unknown, never promise all mandatory fees. Unknown traveler scope requires
price_basis=reference with no travelers, not an assumed adult total. URL
adult=1 versus adult=0 values in historical samples are navigation data, not
pricing evidence. No quote observation, expiry, fare identity, inventory or
revalidation guarantee can be derived from them. A producer may record its
true observation time; otherwise the reference quote is absent. fetched_at
records the current collection batch and does not establish source freshness.

The historical samples departed on 2026-08-22 and establish structure only.
Chinese samples used CNY, English samples USD, and some English results still
had Chinese cabin labels: none proves a general locale/currency rule. Action
URLs are source jump targets, not proof that a selected fare remains available.
Public examples use synthetic dates, identities and example.com URLs and
contain no supplier credentials. Implementers must separately verify actual
calls, hard-condition facts, price/action consistency and supported capabilities.
