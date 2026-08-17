---
name: Book a rate and track the shipment
description: Price a quoted iContainers rate, book it, then poll the resulting booking for status and track-and-trace milestones.
api: openapi/icontainers-brutus-openapi.yml
operations:
  - RateCalculatePrices
  - BookRate
  - GetBookingDetails
  - GetBookingTrackAndTracer
generated: '2026-08-17'
method: generated
source: openapi/icontainers-brutus-openapi.yml
---

# Book a rate and track the shipment

This flow **spends the user's money and creates a real freight booking**. Read the safety rules before
calling `BookRate`.

## Before you start

- You need a rate `uuid` from a quote — see `icontainers-quote-ocean-freight.md`.
- Auth: `Authorization: Bearer <jwt>` on every call.
- **There is no idempotency key in this API.** No operation accepts an `Idempotency-Key` header or
  parameter. A retried `BookRate` after a timeout or a `500` can create a **second booking**.

## Steps

1. **Price the rate first.** Call `RateCalculatePrices`
   (`POST /api/v1/rates/{rateUuid}/calculatePrices`) with the `servicesSelected` booleans (`pickup`,
   `origin`, `destination`, `freight`, `delivery`) and any `optionalBillingItems` selections. The response
   returns the priced `rate` plus `optionalBillingItems` information. Show the user this total **before**
   booking. Never book a rate you have not priced.
2. **Book it.** Call `BookRate` (`POST /api/v1/rates/{rateUuid}/book`) with `currency` (required),
   optional `lang` (`es_ES` or `en_US`), `servicesSelected`, `optionalBillingItems`, `bookingDetails`
   (`cargoReadyDate` — must be today or later — and `commodity` from the closed enum) and optionally
   `addresses` keyed by party type (shipper, consignee, billing).
3. **Expect `202`, not `200`.** The response is `AsyncBooking` = `{ bookingUuid }` and nothing else. The
   booking is not finished when the call returns.
4. **Poll for the outcome.** There are no webhooks and no events in this API, so polling is the only
   option. Call `GetBookingDetails` (`GET /api/v1/bookings/{bookingUuid}/details`) for `status`,
   `totalAmount`, `origin`/`destination` (+ routing ports), `shipmentType`, `items`, `charges` and
   `commodityType`. `status` moves through `PREBOOKING` → `REQUESTED` → `PENDING` → `DONE`, or
   `CANCELED`.
5. **Track the movement.** Call `GetBookingTrackAndTracer`
   (`GET /api/v1/bookings/{bookingUuid}/trackAndTrace`) for `estimatedTimeDeparture`,
   `estimatedTimeArrival`, `actualTimeDeparture`, `actualTimeArrival` and the same `status` enum. `actual*`
   fields are nullable until the milestone happens — a `null` ATA means "not arrived yet", never "unknown".

## Rules

- **Never retry `BookRate` blind.** If the call times out or returns `500`, poll for a booking created in
  the last few minutes (`GetBookingDetails` on any `bookingUuid` you hold, or ask the user to check
  `my.icontainers.com`) before issuing a second booking. Get human confirmation before a second attempt.
- **Confirm the price with the human before step 2.** This is the point of no return in the flow.
- Respect the budget: 60 requests per 60 seconds is shared with your polling. Poll with backoff, not in a
  tight loop, and read `X-RateLimit-Remaining`.
- `403` on a booking means the authenticated account does not own it. Stop; do not retry.
- `422` returns `message` + a per-field `errors` map. Fix the named fields.
- Rate `expirationDate` still applies at booking time — an expired rate will not book.
