---
name: Quote ocean freight with iContainers
description: Search origin and destination places, then create an FCL or LCL ocean freight quote and read its rates from the iContainers Brutus API.
api: openapi/icontainers-brutus-openapi.yml
operations:
  - 34de8aa5b349a2047b75f06e974d840c
  - FclQuote
  - LclQuote
  - GetQuote
generated: '2026-08-17'
method: generated
source: openapi/icontainers-brutus-openapi.yml
---

# Quote ocean freight with iContainers

Use this to turn a shipper's plain-language request ("40ft container, Barcelona to New York, ready in
two weeks") into a real iContainers quote with priced rates.

## Before you start

- Base URL: `https://brutus.icontainers.com` (production) or `https://brutus-dev.icontainers.com`
  (the provider's "Developing server").
- Auth: `Authorization: Bearer <jwt>` on **every** call. There is no self-serve signup — the token comes
  from iContainers (`devsupport@icontainers.com`). A missing or invalid token returns `401` with an empty
  JSON body `{}`.
- Budget: 60 requests per 60 seconds. Watch `X-RateLimit-Remaining` on each response.

## Steps

1. **Resolve the origin and destination.** Call `34de8aa5b349a2047b75f06e974d840c`
   (`GET /api/v1/locations/maritime/places`) once per endpoint of the route with the required `term`
   (free text) and required `shipmentType`. The response is grouped into `cities`, `postalCodes` and
   `seaPorts` — pick the specific object type the quote request expects (port, city or postal code) and
   keep its identifier. Results are **not paginated**; if the term is ambiguous, narrow the term rather
   than paging.
2. **Create the quote.** Call `FclQuote` (`POST /api/v1/quotes/fcl`) for full-container loads or
   `LclQuote` (`POST /api/v1/quotes/lcl`) for less-than-container loads. Send the resolved origin and
   destination location objects plus the cargo items — `ContainerItem` entries for FCL, `LclCargoItem`
   entries for LCL. Declare `DangerousGood` or `TemperatureControlledGoods` when the cargo needs it; the
   commodity enum is closed.
3. **Read the rates.** The quote response carries `uuid`, `quotedAt`, `currency`, `quoteType`,
   `completed` and a `rates` array. Each rate has its own `uuid`, an `expirationDate`, `transitTime`,
   `direct`, `departureWeekdays`, `billingItems`, `suppliersInformation` and a `total`. If `completed` is
   `false` the quote is still being assembled — re-read it with `GetQuote`
   (`GET /api/v1/quotes/{uuid}`) rather than creating a second quote.
4. **Explain the rate to the user using `publicTags`.** Those tags are the provider's own merchandising
   labels: `Promoted`, `Cheapest`, `Fastest`, `Spot`, `Partial`, `Unbreakable`, plus the
   prerequisite/included markers (`Origin Prerequisite`, `Pickup Included`, …). Never invent a reason a
   rate is cheaper — quote the tag.
5. **Hand off.** `quoteOnlineUrl` on the quote points at the same quote in the human UI
   (`my.icontainers.com/quotes/<uuid>`). Give the user that link.

## Rules

- **Watch `expirationDate` on every rate.** Rates expire; a stale rate `uuid` will not book.
- `422` means validation failed. Read `message` and the per-field `errors` map and fix the named fields —
  do not retry the same body. Common causes: `cargoReadyDate` not today or later, `commodity` outside the
  enum.
- `403` means the quote belongs to another account. Do not retry.
- `401`/`403`/`404`/`500` have **no response schema** in the contract, so do not attempt to parse an error
  code out of them — only `422` is machine-readable.
- Quoting is read-only in effect: creating a quote costs nothing and commits nobody. **Booking is not** —
  see `icontainers-book-and-track-shipment.md`.
