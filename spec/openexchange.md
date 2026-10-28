# OpenExchange

Version `0.1.0-draft`.

OpenExchange is the exchange-venue companion to [OpenBook](https://github.com/openbook-data-standards/openbook).
OpenBook models fixtures, markets and odds as documents. It deliberately has no
venue object, no order-book depth and no back/lay semantics. Exchange venues —
Betfair, Matchbook, Smarkets, BETDAQ, and prediction exchanges such as Kalshi,
Polymarket and SX Bet — expose exactly those concepts, and adapters that swallow
them lose the data a consumer actually needs to price a route.

OpenExchange fills that gap. It defines a small set of documents an exchange
adapter emits alongside, or embedded under `x_` fields of, OpenBook documents.

Where prose and schema disagree, the schema wins. Machine-normative material
lives under `schema/*.schema.json` (JSON Schema draft 2020-12).

## Documents

### venue.schema.json — the exchange descriptor

One per upstream. Declares the venue's identity, kind
(`betting_exchange`, `prediction_exchange`, `sportsbook`, `broker`),
jurisdiction, supported currencies, and its `feeProfile` reference.
Venues are never merged: two brands sharing one order book (a distributor)
are one venue with `distribution[]` entries, not two venues.

### market_book.schema.json — exchange market state

The exchange-native state of one market: `status`
(`inactive`, `open`, `suspended`, `closed`, `settled`, `void`),
`inplay`, `betDelay` (seconds), `crossMatching`, `bsp` availability,
matched volume, and the runner list. This is the record that distinguishes
*unavailable, stale, suspended and empty* — a distinction OpenBook market
statuses alone cannot carry.

### runner.schema.json — a selection/runner

`selectionId` (native, required), `name`, `role`
(`win`, `place`, `yes`, `no`, `home`, `away`, `draw`, `over`, `under`,
`participant`), `status` (`active`, `removed`, `winner`, `loser`, `placed`,
`hidden`), `handicap`/`adjustmentFactor`, and `bsp` fields
(`nearPrice`, `farPrice`, `actualSP`). `removalDate` and
`reductionFactor` preserve non-runner deductions.

### order_book.schema.json — depth

Per-runner price levels. `back[]` and `lay[]` are sorted price ladders:
`{price, size}` where `price` is decimal odds strictly greater than 1 and
`size` is a positive decimal string in the venue's stake unit. `side` values
are exchange semantics: `back` means the best price at which the venue will
let you buy the outcome; `lay` the price to sell it. Prediction venues with
YES/NO books map yes-bids to `back` on the YES runner and no-bids to `lay`
(no complement-price derivation — record what the venue reported, flag
synthesized levels with `derived: true`).

`virtualPrice` marks cross-matched/synthetic levels a venue injects
(Betfair `priceMode` aggregated levels, matching-engine mirrors).

### fee_profile.schema.json — venue costs

`commission` model: `net_winnings` (exchange commission on market profit),
`per_contract`, `none`, `other`; `marketBaseRate`, `discountRate`,
`chargeableBasis`; fixed fees (`perContract`, `perTicket`) and payout
units. An all-in quote needs this or it is not comparable across venues.

## Conventions shared with OpenBook

- `identifier[]` on any document that accepts it, `{propertyID, value}`.
- Unknown venue fields belong in `x_<venue>_*`, never dropped.
- Money and sizes are exact decimal strings, never floats.
- Prices are decimal odds > 1. Zero or one are never sentinel values.
- Timestamps are RFC 3339 with timezone.
- `observedAt` records observation time; `publishedAt`/`asOf` records the
  venue's own timestamp. Never substitute one for the other.

## Relationship to OpenBook and OpenVenue

OpenBook documents carry fixture/market/odds identity. An exchange adapter
emits OpenBook documents for the representable surface and OpenExchange
documents (or `x_<venue>_book` blocks shaped like them) for depth, runners,
BSP and fees. [OpenVenue](https://github.com/openbook-data-standards/openvenue)
remains the prose-level reuse target for venue capability descriptors;
OpenExchange is its machine-normative counterpart.
