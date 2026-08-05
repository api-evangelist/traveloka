---
name: traveloka-search-to-booking
description: >-
  Search Traveloka accommodation inventory and complete a booking through the Traveloka Partners
  Network (LOKA) v2 API, from access token to confirmed booking, with the rate re-validation and
  duplicate-prevention steps the provider requires.
api: Traveloka Partners Network (LOKA) v2 Accommodation API
base_url: https://api.travelokapartnersnetwork.com/v2
spec: openapi/traveloka-loka-partner-api-openapi.yml
generated: '2026-08-05'
method: generated
source: >-
  openapi/traveloka-loka-partner-api-openapi.yml +
  https://developer.travelokapartnersnetwork.com/faq
operations:
  - POST /oauth/accesstoken
  - GET /properties/content/hotel
  - GET /properties/content/room
  - GET /properties/:getRates
  - POST /properties/checkRate
  - POST /bookings/booking/create
  - GET /bookings/detail
---

# Search to booking on the Traveloka Partners Network

Every path and parameter below is taken verbatim from the published LOKA v2 OpenAPI (info.version
2.4.8). The published spec assigns an operationId to only one of its eleven operations, so operations
are referenced here by method and path.

## Before you start

- This API is **approval-gated**. You need site credentials from a formalized partnership plus a passed
  certification before you get production access. There is no self-serve signup.
- Staging is `https://api.staging-travelokapartnersnetwork.com/v2`; production is
  `https://api.travelokapartnersnetwork.com/v2`.

## 1. Get an access token

`POST /oauth/accesstoken` on `https://auth-api.afc.traveloka.com/` (staging:
`https://auth-api.afc.staging-traveloka.com/`).

- Body is `application/x-www-form-urlencoded` with `client_id` and `client_secret`.
- Response: `token_type`, `access_token`, `expires_in` (schema `AccessToken`).
- The token lives **60 minutes**. Cache and reuse it — the provider explicitly says not to mint one per
  call. Re-issue only when you see a `401` whose message is `Unauthorized, the token might be expired.`
- Send it on every subsequent call in the `Authorization` header.

## 2. Sync content (cache it, do not poll it)

`GET /properties/content/hotel` and `GET /properties/content/room`.

- `hotelIds` accepts 1–100 ids; `countryISO` filters by ISO 3166-1 alpha-2 country.
- Use `lastUpdatedTime` (ISO 8601) to pull only what changed.
- **Cache for 7 days.** Provider guidance is a weekly refresh, and a full property sync every two weeks.
- Do not run this in the request path of a user search. It is a background sync.

## 3. Search rates

`GET /properties/:getRates`

Required query parameters: `propertyIds`, `checkInDate`, `checkOutDate`, `numRooms`, `numAdults`,
`displayCurrency`, `language`, `userNationality`. Optional: `numChildren`, `childrenAges`, `isExtended`.

Hard published limits — exceeding any of them returns a `422` with an `AFI6xx` code:

| Limit | Value |
|---|---|
| property ids per request | 50 |
| rooms | 8 |
| adults | 30 |
| max child age | 17 |
| length of stay | 15 days |
| booking window | 365 days |

Set `isExtended=true` only when you need `ExtendedRatesResponse` — it returns more per result.

Each result carries a **`rateKey`**. That key is the pivot of the whole flow: it binds one rate to one
occupancy and one stay-date range. Cache rates 15–30 minutes for high-demand properties, 6–12 hours for
low-demand. **Bypass cache entirely inside a live booking flow.**

## 4. Re-validate the rate

`POST /properties/checkRate` with `propertyId`, `roomId`, `rateKey`.

Do this immediately before creating the booking, every time. Skipping it is the documented cause of
`AFI735 MISMATCHED_RATE` / `AFI101 MISMATCH_EXPECTED_RATE` at create time. Rates move between search and
booking; a cached `rateKey` goes stale and returns `AFI080` / `AFI83 RATE_KEY_NOT_FOUND`.

## 5. Create the booking

`POST /bookings/booking/create`

- Include a **unique `partnerBookingId`**. This is the API's idempotency mechanism: Traveloka uses it to
  detect and reject duplicate submissions. Generate it once per booking attempt and persist it *before*
  you send the request.
- Use `partnerNettAmount` as the booking amount.
- Guest details must be consistent with the search spec (adult/child counts and ages) or the booking
  fails validation. `BookingsV2ErrorMessage` is a closed enum of the exact validation strings returned.
- Response: `bookingId`, `partnerBookingId`, `itineraryId`, `propertyId`.

Duplicate signals to handle, not retry through:

| Code | Name |
|---|---|
| AFI734 | BOOKING_ALREADY_EXISTS |
| AFI102 | DOUBLE_CONFIRMATION_ID |
| AFI104 | ALREADY_ISSUED_BOOKING |

## 6. Confirm state

`GET /bookings/detail` with the `bookingId` (or `partnerBookingId`).

On a timeout or a network error during step 5, **do not re-issue the create**. Wait, then poll this
operation to establish the authoritative state. Provider guidance: at most 3 retries, exponential
backoff, 120-second initial delay. A booking stuck in `Pending` for more than 10 minutes is a support
case, not a retry case.

## Error handling

The envelope is `{ data, error: { code, message, requestId } }` — **not** RFC 9457 problem+json. Always
log `error.requestId`; it is the correlation id Traveloka support asks for.

- `400` — parameter missing (`AFI100`).
- `401` — token expired. Re-issue and retry once.
- `404` — property or booking not found.
- `408` — timeout. Back off, then poll rather than repeat a write.
- `422` — validation failure; the `AFI` code names the field.
- `429` — over the rate limit. **100 requests per minute per API key.** Sustained excess trips an "API
  Suspended" state that blocks traffic for 15 seconds.
- `500` — server error. Retry with backoff, quote `requestId` to `partnersnetwork@traveloka.com`.

Full registry of 142 codes: `errors/traveloka-error-codes.yml`.

## The commercial limit you will actually hit first

Traveloka enforces a **Look-to-Book (LTB) ratio** — searches divided by bookings — in the partnership
agreement, separately from the technical rate limit. Exceeding the agreed ratio can trigger suspension
**and a fee**. Practical consequences for how you build:

- Cache per the TTLs above; do not re-search what you already hold.
- Blacklist hotels that are closed, unavailable, or have not converted in three months.
- Do not fan out broad spec multipliers (every length-of-stay × adult-count × room-count combination).
