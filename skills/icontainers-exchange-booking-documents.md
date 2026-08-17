---
name: Exchange booking documents
description: Upload a shipping document against an iContainers booking and download booking or tenant documents by id.
api: openapi/icontainers-brutus-openapi.yml
operations:
  - AddDocumentForBooking
  - GetBookingDocument
  - GetDocument
  - GetDocumentTenant
generated: '2026-08-17'
method: generated
source: openapi/icontainers-brutus-openapi.yml
---

# Exchange booking documents

International freight is a documents business — bill of lading, packing list, commercial invoice, customs
paperwork. This is how the Brutus API moves them.

## Before you start

- You need a `bookingUuid` from `BookRate` or from the user's account.
- `Authorization: Bearer <jwt>` on every call.
- Document ids are **opaque long token strings**, not UUIDs. Never construct or guess one; only use an id
  the API gave you.

## Steps

1. **Upload.** Call `AddDocumentForBooking` (`POST /api/v1/bookings/{bookingUuid}/documents`) with the
   document body. A success is `201` with **no response body** — the contract declares no schema for it, so
   do not try to parse the created document's id out of the response.
2. **List what's attached.** The booking's own payload carries `Document` objects (`id`, `name`,
   `createdAt`). Read them via `GetBookingDetails`
   (`GET /api/v1/bookings/{bookingUuid}/details`) — there is no standalone "list documents" operation.
3. **Download.** Two routes, and they are not interchangeable:
   - `GetBookingDocument` (`GET /api/v1/bookings/{bookingUuid}/documents/{documentId}`) — scoped to a
     booking. Prefer this one; it is the ownership-checked path (it declares `403` for
     "User is not the owner of the booking").
   - `GetDocument` (`GET /api/v1/documents/{documentId}`) — fetch content by id alone.
   - `GetDocumentTenant` (`GET /tenant/v1/documents/{documentId}`) — the same fetch on the `/tenant/v1`
     surface. Only use it if iContainers told you your integration is a tenant integration.

## Rules

- **Never invent a document id.** They are 80+ character opaque tokens; a guessed id is at best a `404`.
- `404` on upload means the booking was not found — verify the `bookingUuid` before retrying.
- `403` means the booking belongs to another account. Stop.
- `422` returns `message` + per-field `errors`.
- Documents can contain a customer's commercial and personal data. Do not echo the contents of a
  downloaded document into a shared transcript, log or summary unless the user asks for it.
- Uploads are writes with no idempotency key — a retried upload can attach the same file twice.
