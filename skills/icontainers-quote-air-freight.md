---
name: Quote air and LTL freight with iContainers
description: Search airports and cities, then create an air freight or less-than-truckload quote and read its rates from the iContainers Brutus API.
api: openapi/icontainers-brutus-openapi.yml
operations:
  - f7652b9cd80cf6ebb58789309b5420bd
  - AirQuote
  - LtlQuote
  - GetQuote
generated: '2026-08-17'
method: generated
source: openapi/icontainers-brutus-openapi.yml
---

# Quote air and LTL freight with iContainers

The air/road sibling of the ocean flow. Same auth, same envelope, different place search and different
cargo shape.

## Before you start

- Base URL `https://brutus.icontainers.com`; `Authorization: Bearer <jwt>` on every call.
- 60 requests per 60 seconds (`X-RateLimit-Limit` / `X-RateLimit-Remaining`).

## Steps

1. **Resolve the route.** Call `f7652b9cd80cf6ebb58789309b5420bd`
   (`GET /api/v1/locations/aerial/places`) with the required `term`. Unlike the maritime search, this one
   takes no `shipmentType` and returns `cities`, `postalCodes` and `airports`. Results are unpaginated.
2. **Create the quote.**
   - `AirQuote` (`POST /api/v1/quotes/air`) for air freight. Cargo is dimensioned: use
     `DimensionedCargoItem` entries with the declared `WeightUnit`, `DimensionUnit` and `VolumeUnit`
     values. Air express and air freight are the same quote surface here.
   - `LtlQuote` (`POST /api/v1/quotes/ltl`) for less-than-truckload road movements.
3. **Read the rates.** Identical shape to ocean: `rates[]` with `uuid`, `expirationDate`, `transitTime`,
   `billingItems`, `suppliersInformation`, `total` and `publicTags`. If the quote response has
   `completed: false`, re-read with `GetQuote` (`GET /api/v1/quotes/{uuid}`) instead of re-quoting.
4. **Continue to booking** with `icontainers-book-and-track-shipment.md` — air, LTL, FCL and LCL all book
   through the same `BookRate` operation on a rate `uuid`.

## Rules

- Declare dangerous goods properly. `DangerousGood` and `TemperatureControlledGoods` are separate schemas
  and the `commodity` enum on a booking includes `Hazardous goods (DGR, IMO)`, `Perishable`,
  `Refrigerated`, `Livestock and Animals` and `Explosives`. Do not map an unclear description onto a
  hazardous class yourself — ask the user.
- `422` returns `message` + per-field `errors`; correct the named fields rather than retrying.
- `401` bodies are empty (`{}`) — treat any `401` as "get a new token", not as a data problem.
- Quotes are free and non-committal; booking is not.
