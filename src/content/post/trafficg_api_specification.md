---
title: "TrafficG REST API Specification (Design Draft)"
description: "Proposed authentication, performance, reservation, payment, queue, and administrative endpoints for TrafficG."
publishDate: 2025-05-10
updatedDate: 2026-10-19
tags: [trafficg, rest-api, system-design]
draft: false
---

> English version of the [original Korean API specification](https://tyge-ottesen-brahe.tistory.com/33). This is a **design document**, not verified deployed API documentation. Original endpoint patterns and response examples are retained with text translated into English; discrepancies are listed at the end.

## Common error codes

| HTTP status | Application code | Description |
|---|---|---|
| 400 | `INVALID_INPUT` | Missing required field or invalid format |
| 401 | `INVALID_CREDENTIALS` | Authentication failed |
| 403 | `FORBIDDEN` | Not authorized |
| 404 | `NOT_FOUND` | Resource does not exist |
| 409 | `CONFLICT` | Duplicate or conflicting resource |
| 429 | `TOO_MANY_ATTEMPTS` | Too many attempts |
| 500 | `SERVER_ERROR` | Internal server error |

## 1. Authentication and account management

### `POST /api/auth/signup`

Request:

```json
{"accountId":"string","password":"string","name":"string","email":"string","phoneNumber":"string","address":"string"}
```

Responses: `201 Created` with `{"userId":123,"message":"Signup successful"}`; `400 INVALID_INPUT` if `accountId` is required; `409 CONFLICT` if the account ID is already in use.

### `POST /api/auth/login`

Request: `{"accountId":"string","password":"string"}`.

Success (`200`): `{"userId":123,"accessToken":"string","refreshToken":"string","message":"Login successful"}`.

Possible errors: `400 INVALID_INPUT` (invalid password format), `401 INVALID_CREDENTIALS`, `429 TOO_MANY_ATTEMPTS` (the original example advises retrying in 15 minutes), and `500 SERVER_ERROR`.

### Other account endpoints

| Method and route | Purpose | Main response |
|---|---|---|
| `POST /api/auth/logout` | Invalidate sign-in state | `204 No Content` |
| `POST /api/auth/refresh` | Refresh access credentials using an `HttpOnly` refresh-token cookie | `200` with new access/refresh tokens |
| `POST /api/auth/checkid` | Check proposed account ID | `200 {"available":true}` or `409 CONFLICT` |

The original logout sketch places `userId`, `accessToken`, and `refreshToken` in a “request header” object. That is an unresolved representation issue; the final authentication contract should use standard authorization/cookie mechanisms and avoid exposing refresh tokens unnecessarily.

## 2. Performance browsing

| Method and route | Response contract |
|---|---|
| `GET /api/shows` | List of `{showId, name, imageUrl, startDate, endDate}`; filters such as `keyword`, `page`, `sort`, and `category` should be query parameters |
| `GET /api/shows/{showId}` | `{showId, name, detail, imageUrl, startDate, endDate}` |
| `GET /api/shows/{showId}/schedules` | List of `{scheduleId, showDate, showTime, ticketingAt}` |
| `GET /api/shows/{showId}/{showScheduleId}/zones` | List of `{zoneId, name, price}` |
| `GET /api/shows/{showId}/{showScheduleId}/{zoneId}/seats` | List of `{seatId, seatNumber, status}` |

The original examples use `2025-05-10` through `2025-05-12` for a performance and a sample schedule at `2025-06-01 19:00` with ticket opening `2025-05-20T10:00:00`. These are illustrative values, not production records.

## 3. Reservation flow

### `POST /api/reservations`

Request:

```json
{"userId":123,"showScheduleId":10,"seatIds":[100,101]}
```

Response: `201 Created`, `{"reservationId":500,"message":"Reserved (pending)"}`.

### `GET /api/reservations/{reservationId}`

Example response:

```json
{
  "reservationId": 500,
  "status": "PENDING",
  "seats": [
    {"seatId":100,"status":"LOCKED"},
    {"seatId":101,"status":"LOCKED"}
  ]
}
```

### Reservation actions

- `POST /api/reservations/{reservationId}/confirm` → `200 {"message":"Reservation confirmed"}`.
- `POST /api/reservations/{reservationId}/cancel` → `200 {"message":"Reservation cancelled"}`.

## 4. Simulated payments

- `POST /api/payments` with `{"reservationId":500,"amount":300000}` → `201` with payment ID 900 and `Payment initiated`.
- `GET /api/payments/{paymentId}` → `200 {"paymentId":900,"status":"PENDING"}`.
- `POST /api/payments/{paymentId}/refund` → `200 {"message":"Refund requested"}`.

These are prototype contracts; they do not demonstrate integration with a real payment service or actual money transfer.

## 5. Waiting queue

- `POST /api/queues/{showScheduleId}/enter` → `201 {"token":"abc123","position":50}`.
- `GET /api/queues/{showScheduleId}/status` → `200 {"token":"abc123","position":45}`.
- `DELETE /api/queues/{showScheduleId}/leave` with queue-token authorization → `204 No Content`.

## 6. Administration

| Method and route | Request fields | Result |
|---|---|---|
| `POST /api/admin/venue` | `name`, `address`, `type` | `201 {"venueId":7}` |
| `POST /api/admin/hall` | `venueId`, `name` | `201 {"hallId":21}` |
| `POST /api/admin/zone` | `hallId`, `name`, `totalSeat` | `201 {"zoneId":42}` |
| `POST /api/admin/seat` | `zoneId`, `seatNumber`, `rowNumber`, `colNumber` | `201 {"seatId":1001}` |
| `POST /api/admin/shows` | `name`, `venueId`, `hallId`, `startDate`, `endDate`, `schedules` | `201 {"showId":333}` |
| `PUT /api/admin/shows/{showId}` | Same shape as show creation | `200 {"message":"Show updated"}` |
| `GET /api/admin/shows/{showId}/status` | Path ID | `200` with `showId`, `totalReserved`, `totalBooked`, `totalWaiting` |

For show creation, the original example includes a schedule array containing `showDate`, `showTime`, and `ticketingAt`. The statistics response example uses show ID 333, 120 reserved, 100 booked, and 20 waiting; these numbers are illustrative.

## Technical review before implementation

1. The feature specification and API specification use different routes for the same operations; choose one canonical contract.
2. `GET` filters should be query parameters rather than JSON request bodies.
3. `userId`, `amount`, role, and reservation ownership must be derived or validated server-side, not trusted solely because they appear in a request.
4. Queue tokens must be treated as authorization credentials with expiry and replay protections.
5. Define consistent error envelopes, idempotency keys for repeatable actions, and safe transaction boundaries for seat holds and confirmation.
6. Avoid promising a payment refund workflow if the project only implements simulated payment confirmation.

The purpose of this specification was to identify how client and server components would communicate before building them.