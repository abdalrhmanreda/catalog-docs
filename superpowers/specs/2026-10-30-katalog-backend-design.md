# Katalog Backend — Design

Date: 2026-10-30

## Intent

Katalog lets a person record messages (dua, advice, love, apology,
reassurance) with optional voice, image and video, and have them delivered
later: right away, on a date, on an occasion, or when the person stops
answering check-ins and a trusted contact confirms. Recipients read them in
the app inbox or through a public link.

An Android app already calls this API. **`RAML/RAML.yaml` is the contract:**
paths, request fields, response fields and the response envelope must match
it exactly. `docs/DATABASE_ERD.md` is the data model.

The backend is built from the template at
https://github.com/abdalrhmanreda/backend-setup (Django 6, DRF, SimpleJWT,
PostgreSQL). Template conventions win wherever the RAML is silent.

## Goals

- Every RAML endpoint, under `/api/v1/`, with the RAML request and response
  shapes.
- Every response uses the RAML envelope:
  `{"status_code": int, "data": <payload|null>, "error": null|{"message", "code"?, "details"?}}`.
- Message bodies stored AES-256-GCM encrypted (`body_encrypted`, `body_iv`,
  `body_tag`).
- Email OTP for email verification, password reset and recipient access.
- Inactivity switch: check-ins, missed-check-in limit, grace period, trustee
  confirmation, release of inactivity messages.
- Template apps kept: `core`, `accounts`, `rbac`, `notifications`, `legal`,
  `support`.

## Non-goals

- Google and Microsoft sign-in (removed with their code, settings and fields).
- Template `billing` app (bank transfers, invoices, receipts). Plans are
  informational only.
- SMS, payment gateways, FCM push sending. Device tokens are stored only.
- A web frontend.

## Anti-goals

- Any response shape that differs from the RAML. The Android app breaks.
- Refresh-token rotation. RAML `/auth/refresh` returns only `access`, so a
  rotated refresh token would be lost and the next refresh would fail.
- Plaintext message bodies in the database.
- Releasing inactivity messages while a trustee is set but has not confirmed.

## Decisions (made by the builder; user said "you decide")

### Platform
- Template versions: Django 6.0, DRF 3.18, SimpleJWT 5.5, psycopg 3.
  `drf-spectacular` added (it was in the original `requirements.txt`) for
  `/api/v1/schema/` and `/api/v1/docs/`.
- Code lives at the repository root (`Catalog/`), next to `RAML/` and `docs/`.
- Every route accepts a path with or without a trailing slash.

### Users
- Custom user model `accounts.User` (UUID primary key, login by email, no
  `username`). Fields follow the ERD `User` table. The template `UserProfile`
  is removed; template code that used it reads `User` fields instead.
- All migrations are regenerated from scratch (the swapped user model
  requires it). This is a new database.
- `ROTATE_REFRESH_TOKENS = False`. Refresh token is returned in the body
  (`AUTH_REFRESH_IN_BODY = true`) and read from the body.
- Login and `POST /checkins/respond` update `last_active_at`.
- `DELETE /me` sets `deleted_at`, `is_active=False`, `status` unchanged,
  renames email to `deleted+<id>@invalid.local`, soft-deletes contacts and
  messages, deletes stored media files and blacklists refresh tokens.
- PIN stored with Django password hashing in `pin_hash`. `has_pin` in the
  User response.

### OTP
- Template `VerificationCode` serves as the ERD `OTPCode`. Purposes:
  `verify_email`, `reset_password`, `recipient_access`.
- `/auth/otp/send` always answers the same 200, whether or not it sent a
  code (no account enumeration). It sends only when: `verify_email` for an
  existing unverified user; `reset_password` for an existing active user;
  `recipient_access` for an email that has a contact with a released
  delivery.
- `/auth/otp/verify`: `verify_email` sets `email_verified_at`;
  `reset_password` spends the code and returns `reset_token`;
  `recipient_access` checks the code without spending it (wrong guesses
  still count). `/r/{token}/verify` spends it.
- `/password/reset` accepts in `code` either the 6-digit OTP or the
  `reset_token`. On success all refresh tokens of the user are blacklisted.

### Plans
- New small `plans` app with the ERD `Plan` table. `seed_plans` creates
  `free`, `monthly`, `one_time`, `business` (placeholder limits, editable in
  Django admin). New users get `free`.
- `max_messages` counts the user's non-draft, non-deleted messages and is
  checked on `/messages/{id}/confirm` (400 `plan_limit_reached`).
- `max_media_mb` is the user's total media storage, checked on
  `/media/upload-url` (400 `storage_limit_reached`).

### Contacts
- At most one `is_self` contact per owner. Its `linked_user` is the owner.
- `linked_user` is set when the contact email matches an active user, on
  contact create/edit and when a user registers.
- Delete is soft (`deleted_at`).
- `/contacts/recent`: up to 10 contacts with `last_used_at`, newest first.
  `last_used_at` is set when a message to that contact is confirmed.
- `/contacts/import`: up to 500 items; an item whose email or phone matches a
  live contact of the owner is skipped.

### Messages
- Body encrypted on every write with AES-256-GCM. Key from env
  `MESSAGE_ENCRYPTION_KEY` (base64, 32 bytes). Without DEBUG, startup fails
  if it is missing. The owner sees the decrypted body.
- Life cycle: `draft` → `confirm` → `scheduled` → released → `sent` → first
  recipient opens → `opened`. `pause` (`scheduled`→`paused`) and `resume`
  (`paused`→`scheduled`). Edit allowed in `draft`, `scheduled`, `paused`.
- `instant` messages release on confirm. `date` and `occasion` need a future
  `deliver_at` and release when it passes. `inactivity` messages release when
  the switch is released.
- `Delivery` rows are created at release, one per live recipient contact.
  Each gets a random 32-byte URL-safe token; only its SHA-256 hex is stored.
  - `email` channel: email with link `{FRONTEND_BASE_URL}/r/{token}`;
    delivery `sent`, or `failed` with `last_error`.
  - `in_app` channel: delivery `delivered`; if the contact has a
    `linked_user`, that user gets a notification.
- Release runs in `messaging.services.release_due_messages()`. It is called
  on the request paths that show messages (inbox, home, message list/detail,
  `/r/{token}`) and by the `process_due` command.
- `/message-types` is a static list in code with English and Arabic labels,
  chosen by `Accept-Language`.

### Media
- Files stored under `PRIVATE_MEDIA_ROOT/media/<user id>/<media id>`.
- `/media/upload-url` creates a `MessageMedia` row (owner = caller, no
  message yet) and returns `upload_url` =
  `<absolute>/api/v1/media/<id>/upload`, which accepts `PUT` with the raw
  bytes and the caller's bearer token. Not in the RAML; Android uses the URL
  it is given.
- `url` in the media response is a signed download link
  (`/api/v1/media/<id>/file?sig=...`) valid for 1 hour.
- Allowed MIME types: `audio/*` for voice, `image/*` for image, `video/*`
  for video. Upload must match `size_bytes` declared.

### Recipients
- `/inbox`: deliveries whose contact `linked_user` is the caller.
- `/r/{token}`: `requires_verification` is true when the contact has an
  email. `/r/{token}/verify` returns `view_session_token` (signed, 30
  minutes). `/r/{token}/open` and `/later` need it in the body
  (`view_session_token`) or header `X-View-Session` when verification is
  required.

### Inactivity
- One `InactivitySettings` row per user, created on first `GET /inactivity`
  (defaults: disabled, 30 days, 3 missed, 7 grace days, verify_by_email
  true, state `active`).
- Evaluation (`inactivity.services.evaluate(settings_row, now)`) on
  `GET /inactivity` and in `process_due`:
  - When `now >= next_checkin_at`: create `Checkin` (channel `email` if
    `verify_by_email` else `in_app`), send email and/or notification, move
    `next_checkin_at` forward by the interval, state `warning`.
  - A check-in counts as missed once it has gone one full interval
    unanswered (sent at T, missed from T + interval). When missed check-ins
    since the last response reach `missed_checkins_limit` (decided before
    any further check-in is sent): with a trustee, state `pending_trustee` and a
    `TrusteeConfirmation` with an emailed link
    `{FRONTEND_BASE_URL}/trustee/{token}`; without a trustee, release when
    `grace_period_days` have passed since the limit was reached.
  - Released: state `released`, user `status = deceased_confirmed`,
    inactivity messages in `scheduled` release.
  - Trustee `rejected`: state `active`, counters reset.
  - `enabled=false`: state `cancelled`. Enabling again: state `active`,
    `next_checkin_at = now + interval`.
- `POST /checkins/respond`: answers open check-ins, state `active` (unless
  `released`), `next_checkin_at = now + interval`.

### Scheduling
- `python manage.py process_due` releases due messages and evaluates every
  enabled inactivity row. A systemd timer runs it every 5 minutes. Request-time
  evaluation keeps the app correct if the timer is late.

### Template apps
- `notifications`: template model kept. API serializer maps `message`→`body`,
  `meta`→`data`. RAML adds `PATCH /notifications/{id}/read`.
- `support`: kept (categories, tickets, admin helpdesk) under
  `/api/v1/support/`. RAML `GET /faq` reads `SupportFAQ`. `BusinessLead` model
  and admin list/update endpoints live in `support`.
- ERD `AuditLog` is the template `rbac.AuditLog`.
- All ids in responses are strings.

## Left to the builder

Admin console screens, email wording, exact seed plan limits, log messages.
