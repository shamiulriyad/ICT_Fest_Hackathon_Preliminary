# CoWork — Coworking Space Booking API

CoWork is a REST API for managing bookable rooms inside a coworking space across
multiple tenant organizations. Each organization has its own rooms, staff
(admins), and members. Members book rooms for time slots; admins manage rooms and
pull reports.

## Stack

- Python 3.11, FastAPI, SQLAlchemy, SQLite (single file, no external DB service)
- JWT auth (access + refresh tokens), HS256, secret from the `JWT_SECRET` env var
- One container, served on **port 8000**

## Setup

```bash
docker compose up --build
```

The database schema is created automatically on first startup — no manual
provisioning or seed scripts. The API listens on `http://localhost:8000`.

To run the smoke test locally:

```bash
pip install -r requirements.txt
pytest
```

## Business rules

1. **Datetimes.** All API datetimes are ISO 8601. Input datetimes carrying a UTC
   offset are converted to UTC before storage or comparison; naive input is
   treated as UTC. All response datetimes are UTC with an explicit UTC designator
   (`Z` or `+00:00`).
2. **Booking price.** `price_cents = hourly_rate_cents × duration_hours`. Duration
   must be a whole number of hours, minimum 1, maximum 8. `end_time` must be
   strictly after `start_time`. `start_time` must be strictly in the future at
   request time — no grace window of any size.
3. **No double-booking.** Two `confirmed` bookings for the same room overlap iff
   `existing.start_time < new.end_time AND new.start_time < existing.end_time`.
   Back-to-back bookings (one ending exactly when the other starts) are allowed.
   Conflict → `409 ROOM_CONFLICT`. Holds under concurrent requests.
4. **Booking quota.** A member may hold at most 3 `confirmed` bookings with
   `start_time` in the window `(now, now + 24h]`, across all rooms in their org.
   Violation → `409 QUOTA_EXCEEDED`. Holds under concurrent requests.
5. **Rate limit.** `POST /bookings` is limited to 20 requests per rolling 60
   seconds per user (all requests count, successful or not). Excess →
   `429 RATE_LIMITED`. Holds under concurrent requests.
6. **Cancellation refund policy.** Only the booking's owner or an admin of the
   same org may cancel. Notice = `start_time − cancellation_time`:
   - notice ≥ 48 hours → 100% refund
   - 24 hours ≤ notice < 48 hours → 50% refund
   - notice < 24 hours → 0% refund

   Refund amount = percentage of `price_cents`, rounded to the nearest cent with
   half-cents rounding up (e.g. 50% of 1001 = 501). Cancelling an
   already-cancelled booking → `409 ALREADY_CANCELLED`. A cancelled booking has
   exactly one RefundLog entry, and the amount returned by the cancel response
   equals the amount stored in the RefundLog. Holds under concurrent cancel
   requests for the same booking.
7. **Reference codes.** Every booking's `reference_code` is unique, including
   under concurrent creation.
8. **Auth.** Tokens are JWTs (HS256) with claims `sub` (user id, string), `org`
   (org id), `role`, `jti` (unique per token), `iat`, `exp`, `type`
   (`access` | `refresh`). Access tokens: `exp − iat` = exactly 900 seconds.
   Refresh tokens: 7 days. Logout immediately invalidates the presented access
   token for all further use (subsequent use → `401`). Refresh tokens are
   single-use: `POST /auth/refresh` returns a new access **and** refresh token and
   invalidates the presented refresh token (reuse → `401`).
9. **Multi-tenancy.** A user (including admins) may only ever read or act on data
   (rooms, bookings, reports, exports, availability, stats) belonging to their own
   organization, on every code path. Cross-org resource IDs behave as
   non-existent → `404`.
10. **Booking visibility.** Members may read and cancel only their own bookings
    (another member's booking id → `404 BOOKING_NOT_FOUND`). Admins may read and
    cancel any booking in their org.
11. **Pagination & ordering.** `GET /bookings` takes `page` (int ≥ 1, default 1)
    and `limit` (int 1–100, default 10). Items are the caller's own bookings
    sorted by ascending `start_time` (ties by ascending `id`). Page N with limit L
    returns items `[(N−1)·L, N·L)` of that ordering; sequential pages never skip
    or repeat items. Response includes `total`.
12. **Usage report.** `GET /admin/usage-report?from=YYYY-MM-DD&to=YYYY-MM-DD`
    returns, per room in the caller's org (including rooms with zero bookings),
    the count of `confirmed` bookings with `start_time` on a date in `[from, to]`
    (UTC, inclusive) and their summed `price_cents`. Cancelled bookings are
    excluded. The report reflects the current state immediately.
13. **Availability.** `GET /rooms/{id}/availability?date=YYYY-MM-DD` returns the
    room's `confirmed` bookings starting on that UTC date as busy intervals,
    sorted ascending. Reflects the current state immediately.
14. **Room stats.** `GET /rooms/{id}/stats` returns the room's current count of
    `confirmed` bookings and their summed `price_cents` (cancellation decrements
    both). Always equals the values derivable from the bookings themselves.
15. **Registration.** `POST /auth/register` with an unknown `org_name` creates the
    org and the user as `admin`; with a known `org_name` it joins the caller as
    `member`. A duplicate username within the org → `409 USERNAME_TAKEN`.
16. **Liveness.** The service responds to all endpoints at all times; no
    combination of concurrent valid requests may hang the service.

## API contract

### Endpoints

| Method | Path | Auth | Success | Description |
|---|---|---|---|---|
| POST | `/auth/register` | No | 201 | Register org admin or join org as member |
| POST | `/auth/login` | No | 200 | Returns access + refresh token |
| POST | `/auth/refresh` | No (refresh token in body) | 200 | Rotates tokens |
| POST | `/auth/logout` | Yes | 200 | Invalidates presented access token |
| GET | `/rooms` | Yes | 200 | List rooms in caller's org |
| POST | `/rooms` | Yes (admin) | 201 | Create a room |
| GET | `/rooms/{id}/availability` | Yes | 200 | Busy intervals for a date |
| GET | `/rooms/{id}/stats` | Yes | 200 | Live confirmed-booking count & revenue |
| POST | `/bookings` | Yes | 201 | Create a booking |
| GET | `/bookings` | Yes | 200 | Caller's bookings, paginated |
| GET | `/bookings/{id}` | Yes | 200 | Single booking incl. refunds |
| POST | `/bookings/{id}/cancel` | Yes | 200 | Cancel + refund calculation |
| GET | `/admin/usage-report` | Yes (admin) | 200 | Per-room usage/revenue for range |
| GET | `/admin/export` | Yes (admin) | 200 | Bookings CSV; `room_id`, `include_all` |
| GET | `/health` | No | 200 | `{"status": "ok"}` |

### Request/response schemas (exact field names)

- `POST /auth/register` body `{org_name, username, password}` →
  `{user_id, org_id, username, role}`
- `POST /auth/login` body `{org_name, username, password}` →
  `{access_token, refresh_token, token_type: "bearer"}`; bad credentials →
  `401 INVALID_CREDENTIALS`
- `POST /auth/refresh` body `{refresh_token}` → same shape as login
- Room: `{id, org_id, name, capacity, hourly_rate_cents}`;
  `POST /rooms` body `{name, capacity, hourly_rate_cents}`
- Availability: `{room_id, date, busy: [{start_time, end_time}, …]}`
- Stats: `{room_id, total_confirmed_bookings, total_revenue_cents}`
- `POST /bookings` body `{room_id, start_time, end_time}` → Booking:
  `{id, reference_code, room_id, user_id, start_time, end_time, status,
  price_cents, created_at}`
- `GET /bookings` → `{items: [Booking, …], page, limit, total}`
- `GET /bookings/{id}` → Booking plus
  `refunds: [{amount_cents, status, processed_at}, …]`
- `POST /bookings/{id}/cancel` →
  `{id, status: "cancelled", refund_percent, refund_amount_cents}`
- Usage report → `{from, to, rooms: [{room_id, room_name, confirmed_bookings,
  revenue_cents}, …]}`
- Export CSV header (exact):
  `id,reference_code,room_id,user_id,start_time,end_time,status,price_cents`

### Errors

Application errors return JSON `{"detail": <string>, "code": <CODE>}` with codes:
`USERNAME_TAKEN` (409), `INVALID_CREDENTIALS` (401), `ROOM_CONFLICT` (409),
`QUOTA_EXCEEDED` (409), `RATE_LIMITED` (429), `ALREADY_CANCELLED` (409),
`BOOKING_NOT_FOUND` (404), `ROOM_NOT_FOUND` (404), `FORBIDDEN` (403),
`INVALID_BOOKING_WINDOW` (400 — past start, non-whole/out-of-range duration, or
`end_time ≤ start_time`). Missing/invalid/expired/blacklisted tokens → 401.
Framework validation errors (422) use FastAPI's default shape.

## Bug fixes

The original implementation shipped with 23 contract-violating bugs (tracked in
[`bug_report.md`](bug_report.md)). All 23 are fixed. Each entry below states what
the bug was, why it produced incorrect behavior, and how it was fixed. Where a
single HTTP request/response can demonstrate the fix, the matching screenshot
from [`output/`](output/) (captured against the live Docker container via
Swagger UI) is included. Concurrency and negative-path bugs can't be shown in
one screenshot, so those are marked as verified by automated test instead.

### Auth (`/auth/register`, `/auth/login`, `/auth/refresh`, `/auth/logout`)

#### 1. Access tokens lived for 15 hours instead of 900 seconds
- **Bug:** `create_access_token()` built the token lifetime as
  `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)`. With
  `ACCESS_TOKEN_EXPIRE_MINUTES = 15`, that's 900 *minutes*, not 900 seconds.
- **Why it was wrong:** the contract requires `exp − iat` to be exactly 900
  seconds. Tokens were living 60x longer than intended, so a logged-out or
  compromised token stayed valid for 15 hours instead of 15 minutes.
- **Fix:** `app/auth.py` now builds the lifetime as
  `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)`. Verified `exp - iat == 900`
  by decoding a live token.

<img src="output/Log In & Obtain Access Token.png" alt="POST /auth/login issuing an access_token/refresh_token pair" width="700">

#### 2. Logout did not revoke the presented access token
- **Bug:** `revoke_access_token()` stored the token's `jti`, but
  `get_token_payload()` checked `payload.get("sub") in _revoked_tokens` — it
  was comparing the revoked-`jti` set against the user id, which never matches.
- **Why it was wrong:** a logged-out access token remained fully usable,
  violating the "logout immediately invalidates the token" rule.
- **Fix:** `get_token_payload()` in `app/auth.py` now checks the token's own
  `jti` against `_revoked_tokens`, guarded by a lock since requests are served
  from a thread pool. Verified: call `/auth/logout`, then reuse the same token
  → `401`. (No single screenshot for this one — it's a two-step sequence — but
  covered by the automated test suite.)

#### 3. Refresh tokens were reusable
- **Bug:** `POST /auth/refresh` decoded a refresh token and issued a new pair
  without ever recording that the presented refresh token had been consumed.
- **Why it was wrong:** the contract requires refresh tokens to be single-use;
  reusing one should return `401`. A stolen refresh token could be replayed
  indefinitely.
- **Fix:** added a locked `_used_refresh_tokens` set and
  `consume_refresh_token()` in `app/auth.py`, called from
  `POST /auth/refresh` before issuing new tokens. Verified: refreshing twice
  with the same refresh token → first call `200`, second call `401`.

#### 4. Duplicate registration returned the existing user instead of `409 USERNAME_TAKEN`
- **Bug:** when a username already existed in the organization,
  `POST /auth/register` returned the existing user's data with a `201`.
- **Why it was wrong:** the contract requires a duplicate username within an
  org to fail with `409 USERNAME_TAKEN`, not silently succeed with someone
  else's account data.
- **Fix:** `app/routers/auth.py` now raises
  `AppError(409, "USERNAME_TAKEN", ...)` when `existing is not None`.

<img src="output/Register a New Organization & Admin.png" alt="POST /auth/register creating the org and returning role: admin" width="700">

### Time handling

#### 5. Datetime offsets were dropped instead of converted to UTC
- **Bug:** `parse_input_datetime()` called `dt.replace(tzinfo=None)` directly
  on an offset-aware input, discarding the offset instead of applying it.
- **Why it was wrong:** `2030-01-01T10:00:00+05:00` was stored as `10:00 UTC`
  instead of `05:00 UTC` — a 5-hour error that corrupted comparisons, prices,
  availability, and every report built on top of it.
- **Fix:** `app/timeutils.py` now calls
  `dt.astimezone(timezone.utc).replace(tzinfo=None)` for aware datetimes.
  Verified with a `+05:00` input booking whose stored `start_time` correctly
  shifted by 5 hours to UTC.

### Booking creation (`POST /bookings`)

#### 6. Past bookings were allowed inside a five-minute grace window
- **Bug:** the past-start check only rejected
  `start <= now - timedelta(seconds=300)`.
- **Why it was wrong:** the contract requires `start_time` to be *strictly* in
  the future with **no grace window of any size**.
- **Fix:** `app/routers/bookings.py` now rejects `start <= now` directly.

#### 7. Invalid booking durations were accepted
- **Bug:** duration validation checked whole-hour and maximum-duration but
  never enforced the 1-hour minimum or that `end_time` is strictly after
  `start_time`.
- **Why it was wrong:** zero-duration and negative-duration bookings could be
  created, producing nonsensical (zero or negative) prices.
- **Fix:** added an explicit `end <= start` rejection and changed the range
  check to `MIN_DURATION_HOURS <= duration_hours <= MAX_DURATION_HOURS`.

#### 8. Back-to-back bookings were incorrectly treated as conflicts
- **Bug:** `_has_conflict()` used `b.start_time <= end and start <= b.end_time`
  — non-strict comparisons that flag a booking ending exactly when another
  starts as overlapping.
- **Why it was wrong:** the contract explicitly allows back-to-back bookings
  and defines overlap as `existing.start < new.end AND new.start < existing.end`.
- **Fix:** switched to strict `<` comparisons.

#### 9. Double-booking protection was not concurrency-safe
#### 10. Booking quota enforcement was not concurrency-safe
- **Bug:** the conflict check and the quota check each did a read, then the
  booking was inserted later with no lock, transaction guard, or DB
  constraint between the check and the insert.
- **Why it was wrong:** concurrent requests could all observe "no conflict" /
  "under quota" and all commit, producing overlapping bookings or more than 3
  confirmed bookings in a 24h window.
- **Fix:** wrapped the conflict check, quota check, and insert in a single
  `threading.Lock()` critical section in `app/routers/bookings.py`, so
  concurrent creation requests are fully serialized. Verified with 10
  concurrent requests for the identical slot → exactly one `201`, the rest
  `409 ROOM_CONFLICT`; and 8 concurrent quota-eligible bookings → at most 3
  succeed.

<img src="output/Book the Room.png" alt="POST /bookings creating a confirmed booking with correct pricing" width="700">

### Reference codes, rate limiting

#### 11. Rate limiting was not thread-safe
- **Bug:** the per-user request bucket was read, trimmed, appended to, and
  written back with no synchronization.
- **Why it was wrong:** concurrent requests could lose updates to the bucket,
  letting more than 20 requests through in a rolling 60-second window.
- **Fix:** `app/services/ratelimit.py` now guards bucket mutation with a
  `threading.Lock()`. Verified with 25 concurrent booking requests from one
  user → some correctly rejected with `429 RATE_LIMITED`, at most 20 let
  through.

#### 12. Reference codes were not guaranteed unique under concurrency
- **Bug:** `next_reference_code()` read a shared counter, then wrote it back
  with no lock; the `reference_code` column was indexed but not unique.
- **Why it was wrong:** concurrent booking creation could hand out the same
  reference code to two different bookings.
- **Fix:** added a `threading.Lock()` around the counter in
  `app/services/reference.py`, plus `unique=True` on
  `Booking.reference_code` in `app/models.py` as a DB-level backstop.
  Verified with 15 concurrent booking creations across different rooms → all
  15 reference codes distinct.

### Reports, caching (`GET /admin/usage-report`, `GET /rooms/{id}/availability`)

#### 13. Creating bookings left cached usage reports stale
- **Bug:** booking creation invalidated the availability cache for that
  room/date but never invalidated the org's usage-report cache.
- **Why it was wrong:** `GET /admin/usage-report` must reflect confirmed
  bookings immediately; a report fetched before a new booking could keep
  returning stale data after the booking was created.
- **Fix:** `create_booking()` in `app/routers/bookings.py` now also calls
  `cache.invalidate_report(user.org_id)`.

<img src="output/Generate Admin Usage Report.png" alt="GET /admin/usage-report reflecting the current confirmed booking and revenue" width="700">

### Booking listing & detail (`GET /bookings`, `GET /bookings/{id}`)

#### 14. Booking pagination used the wrong ordering, offset, and limit
- **Bug:** the endpoint sorted by `start_time.desc()`, used
  `.offset(page * limit)`, and hardcoded `.limit(10)` regardless of the
  requested `limit`.
- **Why it was wrong:** the contract requires ascending `start_time` (ties by
  ascending `id`), page N starting at `(N-1)·limit`, and the caller's actual
  `limit` to be honored — otherwise pages skip/repeat items and ignore the
  query parameter entirely.
- **Fix:** changed to
  `order_by(Booking.start_time.asc(), Booking.id.asc())`,
  `.offset((page - 1) * limit)`, `.limit(limit)`.

<img src="output/Retrieve Bookings List.png" alt="GET /bookings returning items, page, limit, total per contract" width="700">

#### 15. Members could read other members' bookings in the same org
- **Bug:** `get_booking()` scoped the query by organization but never checked
  that the caller owns the booking — the ownership check existed for
  cancellation but not for detail reads.
- **Why it was wrong:** members may only read their own bookings; another
  member's booking id must behave as missing (`404 BOOKING_NOT_FOUND`).
- **Fix:** added
  `if user.role != "admin" and booking.user_id != user.id: raise AppError(404, ...)`
  to `get_booking()`. Verified: a second member requesting the first member's
  booking id → `404 BOOKING_NOT_FOUND`.

#### 16. Booking detail returned `created_at` as `start_time`
- **Bug:** `get_booking()` serialized the booking, then overwrote
  `response["start_time"]` with `booking.created_at`.
- **Why it was wrong:** callers received the booking's *creation* timestamp
  labeled as its *start* time, silently corrupting the response.
- **Fix:** removed the overwrite; `serialize_booking()`'s `start_time` is left
  intact. The screenshot below shows `start_time` (`2026-07-20T10:00:00`,
  the actual reserved slot) and `created_at` (`2026-07-09T14:31:03…`, when the
  request was made) as two distinct, correct values.

<img src="output/see booking room.png" alt="GET /bookings/{id} showing start_time distinct from created_at" width="700">

### Cancellation & refunds (`POST /bookings/{id}/cancel`)

#### 17. Cancellation refund tiers were wrong
- **Bug:** used `notice_hours > 48` (so exactly 48h notice missed the 100%
  tier) and returned 50% for the `< 24h` case instead of 0%.
- **Why it was wrong:** the contract is `>= 48h → 100%`, `24h–48h → 50%`,
  `< 24h → 0%`; the buggy version shortchanged 48h-exact cancellations and
  over-refunded last-minute ones.
- **Fix:** rewrote as direct `timedelta` comparisons:
  `notice >= timedelta(hours=48)` → 100, `notice >= timedelta(hours=24)` → 50,
  else 0.

#### 18. Refund rounding and the persisted refund amount were inconsistent
- **Bug:** the response used Python's `round()` (banker's rounding), while
  `log_refund()` independently recomputed the amount via
  float dollars → `int()` truncation — two different code paths that could
  produce two different numbers for the same cancellation.
- **Why it was wrong:** the contract requires nearest-cent rounding with
  half-cents rounded **up**, and the response amount must equal the amount
  stored in `RefundLog` exactly.
- **Fix:** `cancel_booking()` computes the amount once with integer half-up
  math — `(price_cents * refund_percent + 50) // 100` — and passes that exact
  value to both the response and `log_refund()`, which now just persists the
  given amount instead of recomputing it (`app/services/refunds.py`).

<img src="output/Cancel a Booking.png" alt="POST /bookings/{id}/cancel returning refund_percent and refund_amount_cents" width="700">

#### 19. Concurrent cancellation could create multiple refund logs
- **Bug:** cancellation checked `booking.status`, wrote a refund log, then
  updated `booking.status` — with no lock, atomic update, or uniqueness
  constraint tying the two together.
- **Why it was wrong:** two concurrent cancel requests could both see
  `confirmed`, both write a refund log, and both succeed — violating "exactly
  one `RefundLog` per cancelled booking" and "second attempt → `409`".
- **Fix:** the entire status-check → status-update → refund-log-insert
  sequence is now inside the same `threading.Lock()` used for booking
  creation (`app/routers/bookings.py`), plus `unique=True` on
  `RefundLog.booking_id` (`app/models.py`) as a DB backstop. Verified with 10
  concurrent cancel requests on the same booking → exactly one `200`, nine
  `409 ALREADY_CANCELLED`, and exactly one `RefundLog` entry afterward.

#### 20. Cancelling bookings left cached availability stale
- **Bug:** cancellation invalidated the usage-report cache but never the
  availability cache for the cancelled booking's room/date.
- **Why it was wrong:** `GET /rooms/{id}/availability` must reflect the
  current state immediately; a cached busy interval could remain visible
  after the booking backing it was cancelled.
- **Fix:** `cancel_booking()` now also calls
  `cache.invalidate_availability(booking.room_id, booking.start_time.date().isoformat())`.

<img src="output/Check Room Availability.png" alt="GET /rooms/{id}/availability returning the current confirmed busy intervals" width="700">

### Room stats (`GET /rooms/{id}/stats`)

#### 21. Room stats were stale and race-prone
- **Bug:** `/rooms/{id}/stats` returned values from an in-memory incremental
  counter (`app/services/stats.py`) with no lock, no rebuild from the
  database, and no persistence across restarts.
- **Why it was wrong:** stats must always equal the values derivable from the
  confirmed bookings themselves, including after concurrent activity or a
  process restart — an in-memory counter can drift from the database.
- **Fix:** removed the in-memory counter entirely. `GET /rooms/{id}/stats`
  (`app/routers/rooms.py`) now computes `COUNT`/`SUM` directly from the
  `bookings` table on every request, so it's always correct by construction.

<img src="output/View Live Room Statistics.png" alt="GET /rooms/{id}/stats matching the actual confirmed bookings in the DB" width="700">

### Admin export (`GET /admin/export`)

#### 22. Admin export leaked or mishandled cross-org room IDs
- **Bug:** with `include_all=true` and a `room_id`, the export queried
  bookings filtered only by `room_id` with no org join — leaking another
  organization's bookings. With `include_all=false`, an out-of-org `room_id`
  silently produced an empty CSV instead of a `404`.
- **Why it was wrong:** the contract requires every code path, including
  admin ones, to scope data to the caller's own org, and cross-org resource
  ids to behave as non-existent (`404`).
- **Fix:** `app/services/export.py` now validates any provided `room_id`
  against `Room.org_id == org_id` up front (raising
  `404 ROOM_NOT_FOUND` if missing/cross-org) and always fetches bookings via
  a `Room` join filtered by the caller's org — the unscoped raw-fetch path was
  removed entirely.

<img src="output/Admin export booking.png" alt="GET /admin/export returning bookings scoped to the admin's own org" width="700">

Raw CSV returned by the export above:

```csv
id,reference_code,room_id,user_id,start_time,end_time,status,price_cents
3,CW-001002,4,7,2026-07-20T10:00:00+00:00,2026-07-20T12:00:00+00:00,confirmed,3000
```

### Notifications

#### 23. Notification locks could deadlock the service
- **Bug:** `notify_created()` acquired `_email_lock` then `_audit_lock`, while
  `notify_cancelled()` acquired `_audit_lock` then `_email_lock` — the reverse
  order.
- **Why it was wrong:** a concurrent create and cancel could each hold one
  lock while waiting for the other, hanging both requests forever and
  violating the "no combination of concurrent valid requests may hang the
  service" liveness rule.
- **Fix:** `app/services/notifications.py` — `notify_cancelled()` now
  acquires the locks in the same order as `notify_created()`
  (`_email_lock` then `_audit_lock`). Verified by firing a concurrent create
  and cancel against the same room and confirming both complete within a
  10-second timeout instead of hanging.

## Grading

**Your fixes must preserve this contract exactly** (paths, status codes, error
codes, JSON field names, JWT claims). Grading is **black-box**: the grader builds
the container and asserts behavior against the business rules and API contract
above by talking to the API only.



---

## Collaborators

* **[Shamiul Islam Riyad](https://github.com/shamiulriyad)** - Core Developer
* **[Sayem Rahman](https://github.com/SayemR0018)** - QA & Bug Tester
* **[Rabbi Islam Emon](https://github.com/iamrabbiislamemon)** - Code Reviewer
