---
name: traveloka-booking-cancellation
description: >-
  Look up an existing Traveloka Partners Network booking and submit a cancellation, including the
  cancellation-state machine and the refundability error codes the provider publishes.
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
  - GET /bookings
  - GET /bookings/detail
  - POST /bookings/cancellation/submit
---

# Cancelling a Traveloka Partners Network booking

## 1. Authenticate

`POST /oauth/accesstoken` with `client_id` and `client_secret` as
`application/x-www-form-urlencoded`. Reuse the token for its full 60-minute life.

## 2. Find the booking

- `GET /bookings` — list bookings, filtered by the query parameters the spec declares.
- `GET /bookings/detail` — full detail for one booking (schema `BookingResponseDetail`), including the
  `CancellationPolicy` and any prior `BookingCancellationResponse`.

Resolve by `bookingId` or by your own `partnerBookingId`. If you get `AFI731` / `AFI404`
`BOOKING_NOT_FOUND`, check that the booking was actually **issued** — an unissued booking is not
retrievable by id.

## 3. Check that cancellation is possible before you submit

Read `CancellationPolicy` off the booking detail. The published failure modes are explicit:

| Code | Name | Meaning |
|---|---|---|
| AFI732 | BOOKING_ALREADY_CANCELLED | Already cancelled — this is not a retry case |
| AFI733 | BOOKING_NOT_REFUNDABLE | The rate booked is non-refundable |
| AFI747 / AFI121 | BOOKING_EXPIRED | The booking has expired |
| AFI745 / AFI122 | BOOKING_ALREADY_PAID / ALREADY_PAID_BOOKING | Already settled |
| AFI736 | PENDING_BOOKING | Still pending; resolve state first |

## 4. Submit the cancellation

`POST /bookings/cancellation/submit` with the `bookingId` (schema `BookingCancellationRequest`) and a
cancellation reason.

The response (`BookingCancellationResponse`) returns a `cancellationId` and a `CancellationStatus`.
Documented states:

- `SUBMITTED` — accepted, not yet complete.
- `COMPLETED` — cancellation finished.
- `FAILED` — cancellation rejected.

`SUBMITTED` is **not** terminal. Poll `GET /bookings/detail` until you observe `COMPLETED` or `FAILED`.
Do not re-submit a cancellation that already returned a `cancellationId` — you will get `AFI704
INVALID_CANCEL_ID` or `AFI732 BOOKING_ALREADY_CANCELLED`.

## 5. Refund availability window

If you are checking refund availability, keep the date range under 7 days. The provider states the
maximum is **6 days, 23 hours and 59 minutes**; a wider range errors.

## Error handling

Same envelope as every LOKA operation: `{ data, error: { code, message, requestId } }`, not RFC 9457.
Log `error.requestId` on every failure. `AFI177` / `AFI715 INVALID_CANCELLATION_REQUEST` means the
request body itself is malformed, not that the booking is uncancellable.

Retry policy: at most 3 attempts, exponential backoff, 120-second initial delay. Never retry a
cancellation submit blindly — read state first.

Full registry: `errors/traveloka-error-codes.yml`.
