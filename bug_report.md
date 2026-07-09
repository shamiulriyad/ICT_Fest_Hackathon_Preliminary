# CoWork Bug Report

Source of truth: `ICT_Fest_Hackathon_Preliminary.pdf`, especially the business rules and API contract in sections 3-5.

This report lists the bugs found in the project, with the affected file/line ranges, why each behavior violates the contract, and the required fix.

## 1. Access tokens live for 15 hours instead of 900 seconds

- Files/lines: `app/auth.py:48-58`, `app/config.py:11`
- Bug: `create_access_token()` builds the lifetime with `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)`. With `ACCESS_TOKEN_EXPIRE_MINUTES = 15`, this produces 900 minutes, not 900 seconds.
- Why it is wrong: The contract requires access tokens to expire in exactly 900 seconds.
- Fix: Use `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)` or compute `exp = iat + 900`.

## 2. Logout does not actually revoke the presented access token

- Files/lines: `app/auth.py:85-98`, `app/routers/auth.py:96-99`
- Bug: `revoke_access_token()` stores `payload["jti"]`, but `get_token_payload()` checks `payload.get("sub") in _revoked_tokens`.
- Why it is wrong: The logged-out access token remains usable after `/auth/logout`, violating the immediate invalidation rule.
- Fix: Check the token `jti` against `_revoked_tokens`, not `sub`.

## 3. Refresh tokens are reusable

- Files/lines: `app/routers/auth.py:81-93`, `app/auth.py:63-75`
- Bug: `/auth/refresh` decodes a refresh token and returns new tokens without recording that the presented refresh token was consumed.
- Why it is wrong: Refresh tokens must be single-use; reusing the same refresh token must return 401.
- Fix: Track used refresh-token `jti` values and reject any refresh token whose `jti` has already been used. Mark the presented refresh token used before returning the rotated pair.

## 4. Duplicate registration returns the existing user instead of `409 USERNAME_TAKEN`

- Files/lines: `app/routers/auth.py:32-43`
- Bug: When a username already exists in the organization, the endpoint returns the existing user's data with the route's success status.
- Why it is wrong: The contract requires duplicate usernames within an org to fail with `409 USERNAME_TAKEN`.
- Fix: Raise `AppError(409, "USERNAME_TAKEN", ...)` when `existing is not None`.

## 5. Datetime offsets are dropped instead of converted to UTC

- Files/lines: `app/timeutils.py:5-14`
- Bug: `parse_input_datetime()` removes `tzinfo` with `dt.replace(tzinfo=None)`.
- Why it is wrong: `2030-01-01T10:00:00+05:00` is stored as `10:00 UTC` instead of being converted to `05:00 UTC`, so comparisons, prices, availability, reports, and responses can be wrong.
- Fix: For aware datetimes, call `dt.astimezone(timezone.utc).replace(tzinfo=None)`. Keep naive datetimes as UTC.

## 6. Past bookings are allowed inside a five-minute grace window

- Files/lines: `app/routers/bookings.py:82-87`
- Bug: The code rejects only `start <= now - timedelta(seconds=300)`.
- Why it is wrong: The contract says `start_time` must be strictly in the future at request time, with no grace window.
- Fix: Reject `start <= now`.

## 7. Invalid booking durations are accepted

- Files/lines: `app/routers/bookings.py:89-94`
- Bug: The duration validation checks whole hours and maximum duration, but it never enforces the minimum of 1 hour or that `end_time` is strictly after `start_time`.
- Why it is wrong: Zero-duration and negative-duration bookings can be created, violating the booking window contract and producing invalid prices.
- Fix: Reject `end <= start`; after computing whole-hour duration, require `1 <= duration_hours <= 8`.

## 8. Back-to-back bookings are incorrectly treated as conflicts

- Files/lines: `app/routers/bookings.py:42-51`
- Bug: `_has_conflict()` uses `b.start_time <= end and start <= b.end_time`.
- Why it is wrong: This treats a booking ending exactly when another starts as overlapping. The contract allows back-to-back bookings and defines overlap as `existing.start < new.end AND new.start < existing.end`.
- Fix: Use strict comparisons: `b.start_time < end and start < b.end_time`.

## 9. Double-booking protection is not concurrency safe

- Files/lines: `app/routers/bookings.py:42-52`, `app/routers/bookings.py:100-117`
- Bug: Conflict detection performs a read, then later inserts the booking without a lock, transaction-level guard, or database constraint.
- Why it is wrong: Concurrent requests can both observe no conflict and both commit confirmed overlapping bookings.
- Fix: Guard booking creation with an atomic critical section or SQLite write transaction for the conflict check plus insert. Recheck the overlap inside that atomic section before committing.

## 10. Booking quota enforcement is not concurrency safe

- Files/lines: `app/routers/bookings.py:55-71`, `app/routers/bookings.py:103-117`
- Bug: `_check_quota()` counts existing bookings before the new booking is inserted, with no lock or atomic transaction around count-plus-insert.
- Why it is wrong: Concurrent requests can all see the same count below 3 and commit more than 3 confirmed bookings in the next 24 hours.
- Fix: Enforce quota inside the same atomic creation section used for conflict checks, and re-count immediately before inserting.

## 11. Rate limiting is not thread-safe

- Files/lines: `app/services/ratelimit.py:18-26`, `app/routers/bookings.py:80`
- Bug: The per-user bucket is read, trimmed, slept on, appended, and written back without synchronization.
- Why it is wrong: Concurrent requests can lose updates and allow more than 20 requests in a rolling 60-second window.
- Fix: Protect bucket mutation with a lock, or use an atomic data structure/transaction. Keep counting all attempts, including failed booking attempts.

## 12. Reference codes are not guaranteed unique under concurrency

- Files/lines: `app/services/reference.py:17-21`, `app/models.py:55`, `app/routers/bookings.py:112`
- Bug: `next_reference_code()` reads a shared counter, sleeps, then writes it back without a lock. The database column is indexed but not unique.
- Why it is wrong: Concurrent booking creation can issue the same reference code to multiple bookings.
- Fix: Protect the counter with a lock and add a database uniqueness guarantee or retry-on-conflict strategy for `Booking.reference_code`.

## 13. Creating bookings leaves cached usage reports stale

- Files/lines: `app/routers/bookings.py:116-122`, `app/routers/admin.py:25-27`, `app/routers/admin.py:60-61`, `app/cache.py:12-22`
- Bug: Usage reports are cached, but booking creation only invalidates availability for the room/date. It does not invalidate the org's usage-report cache.
- Why it is wrong: `GET /admin/usage-report` must reflect the current confirmed bookings immediately. A report fetched before a new booking can be returned unchanged after the booking is created.
- Fix: Invalidate the organization's report cache after creating a booking, or remove report caching.

## 14. Booking pagination uses the wrong ordering, offset, and limit

- Files/lines: `app/routers/bookings.py:127-147`
- Bug: The endpoint sorts by `Booking.start_time.desc()`, uses `.offset(page * limit)`, and hardcodes `.limit(10)`.
- Why it is wrong: The contract requires ascending `start_time`, ties by ascending `id`, and page N to start at `(page - 1) * limit` using the requested `limit`.
- Fix: Use `order_by(Booking.start_time.asc(), Booking.id.asc())`, `.offset((page - 1) * limit)`, and `.limit(limit)`.

## 15. Members can read other members' bookings in the same org

- Files/lines: `app/routers/bookings.py:150-175`
- Bug: `get_booking()` scopes by organization but does not enforce member ownership. The ownership check exists in cancellation but not in detail reads.
- Why it is wrong: Members may read only their own bookings. Another member's booking id must behave as missing with `404 BOOKING_NOT_FOUND`.
- Fix: After loading the booking, if `user.role != "admin"` and `booking.user_id != user.id`, raise `AppError(404, "BOOKING_NOT_FOUND", ...)`.

## 16. Booking detail returns `created_at` as `start_time`

- Files/lines: `app/routers/bookings.py:165-167`
- Bug: `get_booking()` serializes the booking, then overwrites `response["start_time"]` with `booking.created_at`.
- Why it is wrong: The response no longer contains the booking's actual start time, violating the Booking schema.
- Fix: Remove the overwrite and leave `serialize_booking()`'s `start_time` intact.

## 17. Cancellation refund tiers are wrong

- Files/lines: `app/routers/bookings.py:198-207`
- Bug: The code uses `notice_hours > 48` instead of `notice >= 48 hours`, and returns 50 percent for the `< 24 hours` case.
- Why it is wrong: Exactly 48 hours, and even 48 hours plus some minutes, can get only 50 percent. Cancellations with less than 24 hours notice incorrectly get 50 percent instead of 0 percent.
- Fix: Compare `notice` directly with `timedelta(hours=48)` and `timedelta(hours=24)`: `>= 48 -> 100`, `>= 24 -> 50`, otherwise `0`.

## 18. Refund rounding and persisted refund amount are incorrect

- Files/lines: `app/routers/bookings.py:208-210`, `app/services/refunds.py:14-27`
- Bug: The response uses Python `round()`, which is banker's rounding, while `log_refund()` truncates through dollar conversion and `int()`.
- Why it is wrong: The contract requires nearest cent with half-cents rounded up, and the cancel response amount must equal the amount stored in `RefundLog`.
- Fix: Calculate refund cents once with integer half-up math, such as `(price_cents * percent + 50) // 100`, and pass/store that exact amount in both the response and `RefundLog`.

## 19. Concurrent cancellation can create multiple refund logs

- Files/lines: `app/routers/bookings.py:184-214`, `app/services/refunds.py:24-25`, `app/models.py:66`
- Bug: Cancellation checks `booking.status`, writes and commits a refund log, sleeps, then marks the booking cancelled. There is no lock, atomic update, or uniqueness constraint for one refund per booking.
- Why it is wrong: Two concurrent cancellations can both see `confirmed`, both create refund logs, and both commit. The contract requires exactly one `RefundLog` for a cancelled booking and `409 ALREADY_CANCELLED` for later attempts.
- Fix: Perform status check, status update, and refund-log insert in one atomic transaction or critical section. Add a uniqueness guarantee on `RefundLog.booking_id` or otherwise enforce one refund per booking.

## 20. Cancelling bookings leaves cached availability stale

- Files/lines: `app/routers/bookings.py:216-218`, `app/routers/rooms.py:69-100`, `app/cache.py:25-34`
- Bug: Cancellation invalidates the usage report cache but does not invalidate the availability cache for the cancelled booking's room/date.
- Why it is wrong: `GET /rooms/{id}/availability` must reflect current confirmed bookings immediately. A cached busy interval can remain visible after cancellation.
- Fix: On successful cancellation, invalidate availability for `booking.room_id` and `booking.start_time.date().isoformat()`.

## 21. Room stats are stale and race-prone

- Files/lines: `app/routers/rooms.py:103-115`, `app/services/stats.py:15-30`, `app/routers/bookings.py:120`, `app/routers/bookings.py:216`
- Bug: `/rooms/{id}/stats` returns values from an in-memory incremental counter. The counter is not protected by locks, is not rebuilt from the database, and is lost on process restart while SQLite data persists.
- Why it is wrong: Room stats must always equal the current confirmed bookings and revenue, including after concurrent activity.
- Fix: Compute stats from the database on request, or maintain the counter through synchronized atomic updates with a startup rebuild. The simplest contract-safe fix is a scoped DB aggregate over confirmed bookings for the room.

## 22. Admin export leaks or mishandles cross-org room IDs

- Files/lines: `app/routers/admin.py:65-73`, `app/services/export.py:22-29`, `app/services/export.py:32-38`, `app/services/export.py:48-54`
- Bug: When `include_all` is true and `room_id` is provided, `generate_export()` calls `fetch_bookings_raw()`, which filters only by `Booking.room_id` and does not join/filter by the admin's org. When `include_all` is false, a cross-org or missing `room_id` silently produces an empty CSV instead of behaving as a missing room.
- Why it is wrong: The PDF's multi-tenancy rule says users, including admins, may only read data from their own organization on every code path, and cross-org resource IDs must behave as non-existent with `404`. The raw export path can leak another organization's bookings, while the scoped path still fails the required `404 ROOM_NOT_FOUND` behavior for an out-of-org `room_id`.
- Fix: Validate any provided `room_id` against `Room.org_id == admin.org_id` before exporting and raise `AppError(404, "ROOM_NOT_FOUND", ...)` if it is missing or outside the org. Remove the raw unscoped path and always query bookings through a `Room` join filtered by the caller's org.

## 23. Notification locks can deadlock the service

- Files/lines: `app/services/notifications.py:24-35`
- Bug: `notify_created()` acquires `_email_lock` then `_audit_lock`, while `notify_cancelled()` acquires `_audit_lock` then `_email_lock`.
- Why it is wrong: A concurrent create and cancel can each hold one lock and wait forever for the other, hanging requests and violating the liveness rule.
- Fix: Acquire locks in the same order in both functions, or replace nested locks with one lock around the combined notification/audit critical section.
