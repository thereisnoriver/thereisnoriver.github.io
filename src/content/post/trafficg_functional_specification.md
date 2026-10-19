---
title: "TrafficG Functional Specification: Users, Reservations, and Administration"
description: "A design-stage specification covering account management, ticket queues, seat holds, simulated payments, and administrative workflows."
publishDate: 2025-05-05
updatedDate: 2026-10-19
tags: [trafficg, api-design, concurrency, system-design]
draft: false
---

> Translated and reorganized from the [original Korean specification](https://tyge-ottesen-brahe.tistory.com/30). **Design specification only**: endpoints and processing rules below do not necessarily match completed implementation. This draft is a structured translation of the functional requirements, with technical review notes clearly separated.

## User functionality

### Registration and login

**Registration:** Collect an account ID, password, email address, phone number, and relevant verification information. Validate required fields and password policy, check duplicate identifiers, verify the phone number if that feature is enabled, and create the account. The original design listed `POST /api/users/checkAccount`, `POST /api/auth/checkId`, and `POST /api/users/signUp`, with conflict responses for duplicate accounts or emails. On success, redirect to the login page or display a navigation button.

**Login:** Verify account credentials and status; issue access and refresh tokens; return the access token to the client and transmit the refresh token using an `HttpOnly` cookie. The client would include an access token in authorized requests. The initial planned routes were `POST /api/login`, `POST /api/token/refresh`, and optionally `POST /api/logout`. Invalid, missing, or disabled accounts should not gain access.

### Performance browsing

**List:** Show performance names, posters, and dates. The design includes newest-first sorting, deadline-based sorting, popularity sorting, pagination (`page`), search (`keyword`), and category filtering. An example request is `GET /api/shows?page=1&sort=popular&category=musical&keyword=Rebecca`. Empty lists and empty search results need separate user-friendly messages.

**Detail:** `GET /api/shows/{showId}` would return the name, date/time, images, description, and booking opening time. Before sales open, the reservation button remains disabled; a countdown appears near opening time; and the button becomes available at the configured time. Invalid IDs yield a 404, and completed performances show a closed state.

### Waiting room

The goal is to limit how many users enter seat selection simultaneously.

1. A user calls `POST /api/shows/{showId}/waiting/enter` and receives a queue token.
2. The client polls `GET /api/shows/{showId}/waiting/status` at one-second intervals to display progress.
3. The original design considered a display-only queue number that decreases in units of 100, separate from the actual admission decision.
4. When the client appears ready to enter, the server checks authoritative admission conditions, including an active-user threshold of 100 and token validity.
5. Expired or missing queue tokens require reentry; leaving could be represented by `DELETE /api/waiting/{token}`.

**Important review question:** A display-only number must not mislead users about an actual position or become an authorization signal. Admission should be decided atomically by the server, with fair ordering and reliable concurrency control. Polling every second also requires load testing and rate consideration.

### Zone and seat selection

`GET /api/shows/{showId}/zones` returns zone identifiers, prices, total seats, and available seats. After choosing a zone, `GET /api/seats/{showId}?zone={zoneId}` returns seats with row and column coordinates.

The UI distinguishes available seats from booked ones and allows users to select **up to four seats**. At this stage, selection exists only in the browser; it is **not a reservation or lock**. A request that relies on stale availability can still fail later.

### Reservation and seat hold

After selection, the client submits a reservation request. The original spec refers to both `POST /api/seats/lock` and `POST /api/reservation`; the route needs to be made consistent with the separately published API specification.

The proposed backend uses Redis `SETNX`-style conditional acquisition with a five-minute TTL. If every requested seat can be held, the server creates a reservation with status `PENDING`. If any seat is unavailable, the server should return a conflict and tell the client which seat selections need to be reconsidered. Authentication and seat-ID validation are mandatory.

**Technical review:** Holding multiple seats one by one can leave a partial lock set if a later acquisition fails. A design needs atomic or compensating behavior, unique lock ownership, and release checks so one user cannot release another user's hold. A Redis lock is also not a substitute for a durable transactional constraint in the authoritative booking database.

### Simulated payment and confirmation

In the prototype, payment confirmation was represented by a simulated user action rather than a real payment gateway. After checking the confirmation input and the still-valid seat holds, `POST /api/reservations/{reservationId}/confirm` was intended to change the reservation to `CONFIRMED`, mark seats as booked, and release the holds.

Failure cases include incorrect confirmation input, expired holds, duplicate requests, and unauthenticated attempts. The corresponding outcomes include keeping the pending reservation, cancellation, a conflict response, or redirecting to a result page.

**Review:** Confirmation and seat-state changes must be idempotent and coordinated transactionally. A clock-based TTL expiration by itself does not automatically update relational reservation records.

### Automatic cancellation and result view

The original specification proposed a five-minute Redis TTL for `seat-lock:{seatId}`. After expiration, either a payment attempt or a periodic backend job would detect a `PENDING` reservation whose hold was gone and mark it `CANCELLED`, restoring seat availability. A later confirmation attempt would receive an expiration/conflict response.

`GET /api/reservations/{reservationId}` provides the result view. For confirmed reservations, it returns the performance, zone, seats, and confirmation status. Unauthorized access to someone else's reservation must be prevented even when a reservation ID is known.

## Administrative functionality

| Feature | Proposed endpoint | Key validation or behavior |
|---|---|---|
| Administrator login | `POST /api/admin/login` | Verify credentials and administrator role |
| Token renewal | `POST /api/admin/token/refresh` | Validate refresh token and role |
| Register venue | `POST /api/admin/venue` | Reject duplicate name, validate address |
| Register hall | `POST /api/admin/hall` | Venue must exist, hall name unique within venue |
| Register zones and seats | `POST /api/admin/seats` | Validate hall, unique zones, nonempty seats |
| Register performance | `POST /api/admin/shows/insert` | Venue/hall/schedule/zone pricing/opening time |
| Edit performance | `PUT /api/admin/shows/{showId}` | Reject nonexistent resources and invalid fields |
| Booking statistics | Planned for later development | By performance, schedule, zone, and state |

The original administrative workflows specified navigation to the next configuration screen after successful venue, hall, zone, or performance registration. They also included updates and deletions of performances and an as-yet-unimplemented reservation status dashboard.

## Design decisions needing validation

- Align endpoint names and response shapes across the functional and API specifications.
- Never trust a client-provided user ID, price, or reservation amount without server-side authorization and recalculation.
- Define atomic multi-seat reservation semantics and rollback of partial lock attempts.
- Make simulated confirmation idempotent and enforce uniqueness in durable storage.
- Implement authorization checks for queue tokens, reservation IDs, and administrator actions.
- Specify time-zone behavior for performance dates and ticket opening times.

The value of writing the specification first was seeing how a seemingly straightforward ticket reservation UI quickly becomes a concurrency and state-consistency problem.