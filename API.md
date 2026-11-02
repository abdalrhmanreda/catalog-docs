# Catalog API guide

For the Android developers who call this API. It describes what the backend
does today. The contract it was built from is `RAML/RAML.yaml`; where the
code and the RAML differ, this guide follows the code and lists the
difference in [Differences from RAML](#8-differences-from-raml).

Contents:

1. [Overview](#1-overview)
2. [Response envelope](#2-response-envelope)
3. [Authentication](#3-authentication)
4. [Error codes](#4-error-codes)
5. [Rate limits](#5-rate-limits)
6. [Endpoint reference](#6-endpoint-reference)
7. [Flows](#7-flows)
8. [Differences from RAML](#8-differences-from-raml)
9. [Changelog](#9-changelog)

## 1. Overview

| | |
|---|---|
| Base URL | `https://api.catalog.app/api/v1` |
| Local | `http://localhost:8000/api/v1` |
| Format | JSON in, JSON out (`Content-Type: application/json`). The only exceptions are the media upload (raw bytes) and download (the file). |
| Trailing slash | Optional: `/me` and `/me/` are the same endpoint. |
| `X-Platform` header | Required on every request: `ios`, `android` or `web`. Not needed on `/health`, `/schema`, `/docs`, `/redoc` or the signed `/media/{id}/file` URL. A missing or unknown value gets `400 invalid_platform`. |
| Ids | UUID strings, except notification ids, which are integers sent as strings (`"12"`). |
| Dates | ISO-8601 in UTC, for example `2026-10-31T09:00:00Z` or `2026-10-31T09:00:00.123456Z`. Send dates the same way. |
| Swagger UI | `/api/v1/docs` (live server) and [`docs/swagger.html`](swagger.html) (standalone interactive viewer) |
| ReDoc | `/api/v1/redoc` |
| OpenAPI file | `/api/v1/schema` (live), [`docs/openapi.yaml`](openapi.yaml) and [`docs/openapi.json`](openapi.json) in the repository (the same schema, kept in step by a test) |

Empty optional strings come back as `""`, not `null` (`phone`, `avatar_url`,
`relation`, `occasion_note`). Nullable fields come back as `null` and are
marked nullable in the schema.

## 2. Response envelope

Every JSON response, success or error, has the same three keys.

Success:

```json
{
  "status_code": 200,
  "data": { "valid": true },
  "error": null
}
```

Error:

```json
{
  "status_code": 400,
  "data": null,
  "error": {
    "message": "Invalid request data.",
    "code": "validation_error",
    "details": {
      "email": ["Enter a valid email address."],
      "name": ["This field is required."]
    }
  }
}
```

- `status_code` repeats the HTTP status.
- `error.code` is stable: branch on it. `error.message` is English text that
  is safe to show but may change.
- `error.details` appears only on some errors. For `validation_error` it maps
  each field to a list of messages; a problem with the body as a whole is
  under `non_field_errors`.

Lists (`GET /contacts`, `/messages`, `/inbox`, `/notifications`, `/plans`,
`/faq`, `/message-types`, `/contacts/recent`) return a bare array in `data`.
`/contacts`, `/messages`, `/inbox` and `/notifications` also accept
`?page_size=N` (max 200) and then return a page instead:

```json
{
  "status_code": 200,
  "data": {
    "count": 4,
    "next": "https://api.catalog.app/api/v1/contacts?page=2&page_size=2",
    "previous": null,
    "results": [ { "id": "47660aef-75f9-4416-8a34-6c98e244bc6d", "name": "Fatima" } ]
  },
  "error": null
}
```

Use `?page=N` to move between pages. Without `page_size` there is no paging.

The two media file endpoints are the exception to the envelope:
`PUT /media/{id}/upload` takes raw bytes (its response is enveloped JSON), and
`GET /media/{id}/file` returns the file itself.

## 3. Authentication

Register (`POST /auth/register`) and log in (`POST /auth/login`) both return:

```json
{
  "user": { "id": "8f6d6c64-9eda-4ba6-9314-5208f305a825", "full_name": "Sara Ali", "...": "..." },
  "access": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

| Token | Lifetime | Use |
|---|---|---|
| `access` | 15 minutes | `Authorization: Bearer <access>` on every authenticated call |
| `refresh` | 30 days | `POST /auth/refresh` to get a new `access` |

The refresh token is **not rotated**: `/auth/refresh` returns only a new
`access`, and the same `refresh` keeps working until it expires or is
revoked. Store it once.

What the app should do:

1. Send `Authorization: Bearer <access>`.
2. On a `401` with `error.code` `unauthorized`, call `POST /auth/refresh` with
   the stored refresh token **once**, then retry the request with the new
   access token.
3. If `/auth/refresh` itself answers `401` (`unauthorized`), the session is
   over: clear the tokens and send the user to sign in.

Every authentication failure is `401` with code `unauthorized`; the message
says which one (no token, expired or malformed token, revoked refresh
token, account no longer allowed in). Public endpoints ignore the
`Authorization` header, so a stale token never breaks sign-in.

A refresh token is revoked by:

- `POST /auth/logout` (that one token),
- a successful password reset (every token of the account),
- deleting the account (`DELETE /me`).

Accounts that are suspended or deleted cannot sign in: login answers
`invalid_credentials`, and their existing access and refresh tokens get
`401 unauthorized`. An account whose inactivity switch has released
(`status: deceased_confirmed`) can still sign in.

## 4. Error codes

| `error.code` | HTTP | Where | Meaning | What the app should do |
|---|---|---|---|---|
| `validation_error` | 400 | Any endpoint with a body or query rules | A field is missing or invalid. `details` lists the fields. | Show the field messages next to the fields. |
| `invalid_platform` | 400 | Every endpoint except those listed in section 1 | The `X-Platform` header is missing or is not `ios`, `android` or `web`. | Send the header (a client bug). |
| `parse_error` | 400 | Any endpoint with a body | The body is not valid JSON. | Fix the request (a client bug). |
| `invalid_credentials` | 400 | `POST /auth/login` | Unknown email, wrong password, or an account that may not sign in. | Show "email or password is wrong". |
| `invalid_token` | 400 | `POST /auth/logout` | The refresh token is not a live refresh token of this user. | Clear local tokens anyway. |
| `no_code` | 400 | `/auth/otp/verify`, `/auth/password/reset`, `/r/{token}/verify` | There is no active code for this email and purpose (never sent, already used, or replaced). | Offer "send a new code". |
| `expired` | 400 | same | The code is older than 5 minutes (server default, `VERIFICATION_CODE_TTL_MINUTES`). | Offer "send a new code". |
| `invalid` | 400 | same | Wrong code. The message says how many attempts are left. | Let the user try again. |
| `attempts_exhausted` | 400 | same | 5 wrong guesses (server default, `VERIFICATION_CODE_MAX_ATTEMPTS`): the code is spent. | Offer "send a new code". |
| `invalid_reset_token` | 400 | `POST /auth/password/reset` | The `reset_token` is unknown, used or older than 10 minutes (server default, `PASSWORD_RESET_TOKEN_TTL_MINUTES`). | Restart the reset flow. |
| `old_pin_required` | 400 | `POST /me/pin` | A PIN is already set; changing it needs `old_pin`. | Ask for the current PIN. |
| `invalid_pin` | 400 | `POST /me/pin` | `old_pin` is wrong. | Ask again. |
| `self_contact_exists` | 400 | `POST /contacts` | There is already an `is_self` contact. | Use the existing one. |
| `trustee_email_required` | 400 | `PATCH /contacts/{id}`, `PUT /inactivity` | The trusted contact must have an email (removing it, or choosing a contact without one). | Ask for an email, or pick another contact. |
| `contact_is_trustee` | 400 | `DELETE /contacts/{id}` | The contact is the inactivity switch's trusted contact. | Choose another trusted contact first. |
| `file_too_large` | 400 | `POST /media/upload-url`, `PUT /media/{id}/upload` | Larger than the per-file upload limit (100 MB by default). | Tell the user the file is too big. |
| `storage_limit_reached` | 400 | `POST /media/upload-url` | The file would take the user past the plan's `max_media_mb`. | Suggest deleting media or upgrading. |
| `size_mismatch` | 400 | `PUT /media/{id}/upload` | Body length differs from the declared `size_bytes`. | Re-send the exact file. |
| `already_uploaded` | 400 | `PUT /media/{id}/upload` | This media already has its bytes. | Go on to `/confirm`. |
| `not_uploaded` | 400 | `POST /media/{id}/confirm` | The bytes were never uploaded. | Upload first. |
| `invalid_state` | 400 | `PATCH /messages/{id}`, `/confirm`, `/pause`, `/resume`, `DELETE /media/{id}` | Not allowed in the message's current status (see the life cycle in 7.3). | Refresh the message and update the UI. |
| `deliver_at_passed` | 400 | `POST /messages/{id}/confirm` | A date or occasion message whose `deliver_at` is no longer in the future. | Ask for a new date (`PATCH`), then confirm. |
| `no_recipients` | 400 | `POST /messages/{id}/confirm` | Every recipient has been deleted. | Ask for recipients (`PATCH`). |
| `plan_limit_reached` | 400 | `POST /messages/{id}/confirm` | The plan's `max_messages` confirmed messages are used. | Suggest upgrading; the draft is kept. |
| `email_mismatch` | 400 | `POST /r/{token}/verify` | The email is not the one the message was sent to. | Ask for the right email. |
| `verification_required` | 400 | `POST /r/{token}/open`, `/later` | The recipient must verify their email and send the view session. | Run the verify step (7.5). |
| `already_decided` | 400 | `POST /trustee/{token}/decision` | The request was already answered or withdrawn. | Show the current status (`GET /trustee/{token}`). |
| `unauthorized` | 401 | Every authenticated endpoint, `/auth/refresh` | Missing, invalid or expired token, revoked refresh token, or an account that can no longer sign in. | Refresh once; if that fails, sign out (section 3). |
| `email_not_verified` | 403 | Support ticket writes (`/support/tickets...`) | The account's email is not verified. | Ask the user to verify their email. |
| `permission_denied` | 403 | Support tickets of another user, admin endpoints | Not allowed. (Admin endpoints use `PERMISSION_DENIED`.) | Not reachable from normal app use. |
| `not_found` | 404 | Any id or token in the path; unknown URLs in production | Does not exist, is deleted, or is not yours. Expired or wrong `sig` on `/media/{id}/file` too. | Remove it from the UI. |
| `method_not_allowed` | 405 | Any | Wrong HTTP method. | A client bug. |
| `unsupported_media_type` | 415 | Any endpoint with a body | The body's Content-Type is not JSON, form or multipart. | Send `application/json`. |
| `throttled` | 429 | See section 5 | Too many requests. The `Retry-After` header and the message give the seconds to wait. | Wait, then retry. |
| `error` | 503 | `GET /health` | The database is unreachable; `details` is `{"status": "degraded", "database": "unreachable"}`. | Monitoring only. |
| `server_error` | 500 | Any | Unexpected server error (production). | Show a generic error; retry later. |

## 5. Rate limits

Limits count per client IP unless noted. A refused request gets `429`:

```json
{
  "status_code": 429,
  "data": null,
  "error": {
    "message": "Request was throttled. Expected available in 60 seconds.",
    "code": "throttled"
  }
}
```

with a `Retry-After: 60` header.

| Endpoint | Limits (defaults; the server may set others) |
|---|---|
| `POST /auth/register` | 10/hour per IP; 5/hour per email in the body |
| `POST /auth/login` | 10/hour per IP; 20/hour per email in the body, from any IP |
| `POST /auth/otp/send` | 1/minute and 5/hour per IP; 5/hour per `target` email |
| `POST /auth/otp/verify` | 15/hour per IP; 10/hour per `target` email |
| `POST /auth/password/forgot` | 5/hour per IP; 1/minute and 5/hour per email |
| `POST /auth/password/reset` | 10/hour and 15/hour per IP; 10/hour per email |
| `POST /me/pin`, `POST /me/pin/verify` | 10/hour per user, shared by both |
| `POST /r/{token}/verify` | 15/hour per IP; 10/hour per email (both counts shared with `/auth/otp/verify`) |
| `POST /business-leads` | 5/hour per IP |
| Everything else | 300/minute per IP without a token; 1,200/minute per signed-in user |

`GET /health` is never throttled.

## 6. Endpoint reference

Grouped like Swagger, in RAML order. "Auth" says whether the endpoint needs
`Authorization: Bearer <access>`. Every endpoint that needs it can also
answer `401 unauthorized`; that is not repeated below. `data` examples show
only the `data` part of the envelope.

Reusable objects:

**User**

```json
{
  "id": "8f6d6c64-9eda-4ba6-9314-5208f305a825",
  "full_name": "Sara Ali",
  "email": "sara@example.com",
  "phone": "",
  "avatar_url": "",
  "has_pin": false,
  "email_verified_at": "2026-10-31T00:34:15.200244Z",
  "locale": "en",
  "timezone": "Africa/Cairo",
  "last_active_at": "2026-10-31T00:34:15.960780Z",
  "status": "active",
  "created_at": "2026-10-31T00:34:14.041627Z"
}
```

`email_verified_at` is `null` until the email is verified. `status` is
`active`, `suspended` or `deceased_confirmed`.

**Contact**

```json
{
  "id": "a955a992-8aca-460f-980e-992585fce347",
  "owner_id": "8f6d6c64-9eda-4ba6-9314-5208f305a825",
  "name": "Omar",
  "relation": "Brother",
  "email": "omar@example.com",
  "phone": "",
  "is_self": false,
  "linked_user_id": "89889aa6-d71b-4af1-a5c1-e3ae114d4233",
  "last_used_at": null,
  "created_at": "2026-10-31T00:34:17.057220Z"
}
```

`linked_user_id` is set when the contact's email belongs to a verified
Catalog account (in-app messages to this contact reach that account's
inbox), else `null`. `last_used_at` is set when a confirmed message goes to
the contact.

**MessageMedia**

```json
{
  "id": "f0a5ccd9-1829-4bad-bf43-091cf564520a",
  "kind": "voice",
  "storage_key": "media/8f6d6c64-9eda-4ba6-9314-5208f305a825/f0a5ccd9-1829-4bad-bf43-091cf564520a",
  "url": "https://api.catalog.app/api/v1/media/f0a5ccd9-1829-4bad-bf43-091cf564520a/file?sig=f0a5ccd9-...:1xMx33:SJ1P0U...",
  "mime_type": "audio/mp4",
  "size_bytes": 48213,
  "duration_sec": 3,
  "sort_order": 0
}
```

`url` is a signed download link, valid for 1 hour, and `null` until the
bytes are uploaded. Fetch it without the bearer token. Re-read the message to
get a fresh link.

**Message**

```json
{
  "id": "986431a1-aecb-4721-8457-8236a0990743",
  "sender_id": "8f6d6c64-9eda-4ba6-9314-5208f305a825",
  "type": "advice",
  "title": "For your graduation day",
  "body": "I am always proud of you...",
  "status": "scheduled",
  "trigger_type": "date",
  "deliver_at": "2027-06-01T10:00:00Z",
  "occasion_type": null,
  "occasion_note": "",
  "channel": "email",
  "is_saved_moment": false,
  "paused_at": null,
  "confirmed_at": "2026-10-31T00:34:17.189975Z",
  "sent_at": null,
  "created_at": "2026-10-31T00:34:17.181458Z",
  "media": [],
  "recipient_ids": ["47660aef-75f9-4416-8a34-6c98e244bc6d"]
}
```

| Field | Values |
|---|---|
| `type` | `dua`, `advice`, `love`, `apology`, `reassurance` |
| `status` | `draft`, `scheduled`, `paused`, `sent`, `opened` (`queued` and `failed` exist in the enum but are not used for messages today) |
| `trigger_type` | `instant`, `date`, `occasion`, `inactivity` |
| `occasion_type` | `birth`, `wedding`, `engagement`, `graduation`, or `null` |
| `channel` | `in_app`, `email` |

`body` is stored encrypted and returned decrypted to its owner.

### 6.1 Auth

#### POST /auth/register

Auth: no. Creates the account (free plan), emails a `verify_email` code and
signs the user in.

| Field | Type | Required | Rules |
|---|---|---|---|
| `full_name` | string | yes | max 150 |
| `email` | string | yes | valid email, not already registered (case-insensitive) |
| `password` | string | yes | at least 8 characters, not too common, not all digits, not too close to the name or email |
| `phone` | string | no | max 32 |
| `locale` | string | no | max 10, default `en` |
| `timezone` | string | no | IANA name such as `Africa/Cairo`, default `UTC` |

201 `data`: `{ "user": User, "access": "...", "refresh": "..." }` (see section 3).
Errors: `validation_error` (also for a taken email: `details.email`), `throttled`.

#### POST /auth/login

Auth: no.

| Field | Type | Required |
|---|---|---|
| `email` | string | yes |
| `password` | string | yes |

200 `data`: `{ "user": User, "access": "...", "refresh": "..." }`.
Errors: `validation_error`, `invalid_credentials`, `throttled`.

#### POST /auth/refresh

Auth: no.

| Field | Type | Required |
|---|---|---|
| `refresh` | string | yes |

200 `data`:

```json
{ "access": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." }
```

Errors: `validation_error` (missing `refresh`), `unauthorized` (401: expired,
malformed or revoked refresh token, or the account can no longer sign in).

#### POST /auth/logout

Auth: yes. Revokes the refresh token sent, which must belong to the caller.

| Field | Type | Required |
|---|---|---|
| `refresh` | string | yes |

200 `data`: `{ "detail": "Successfully logged out." }`.
Errors: `validation_error`, `invalid_token`.

#### POST /auth/otp/send

Auth: no. Always answers the same 200, whether or not a code was sent, so
the answer never reveals whether an address has an account.

| Field | Type | Required | Rules |
|---|---|---|---|
| `target` | string | yes | email |
| `purpose` | string | yes | `verify_email`, `reset_password`, `recipient_access` |

A code is actually sent only for: `verify_email` to an account whose email is
not verified; `reset_password` to an account that can sign in;
`recipient_access` to an email that has received a released message. A new
code replaces the previous one for that email and purpose.

200 `data`:

```json
{ "message": "OTP sent successfully.", "expires_in_seconds": 300 }
```

Errors: `validation_error`, `throttled`.

#### POST /auth/otp/verify

Auth: no.

| Field | Type | Required | Rules |
|---|---|---|---|
| `target` | string | yes | email |
| `code` | string | yes | 6 digits |
| `purpose` | string | yes | as above |

- `verify_email`: marks the email verified. From then on contacts carrying
  this email are linked to the account and in-app messages reach its inbox.
- `reset_password`: spends the code and returns `reset_token` (valid 10
  minutes) for `/auth/password/reset`.
- `recipient_access`: checks the code but does not spend it;
  `/r/{token}/verify` spends it.

Each wrong guess counts; the fifth spends the code.

200 `data`:

```json
{ "verified": true, "reset_token": "V6uQq6u5H6a2e7Wu-7TKNV-RBrEXpCDlvRr_L8Fbwp4" }
```

(`reset_token` only for `reset_password`.)
Errors: `validation_error`, `no_code`, `expired`, `invalid`,
`attempts_exhausted`, `throttled`.

#### POST /auth/password/forgot

Auth: no. Emails a `reset_password` code if the account exists and can sign
in; the same 200 either way.

| Field | Type | Required |
|---|---|---|
| `email` | string | yes |

200 `data`: `{ "message": "Password reset OTP sent to email." }`.
Errors: `validation_error`, `throttled`.

#### POST /auth/password/reset

Auth: no.

| Field | Type | Required | Rules |
|---|---|---|---|
| `email` | string | yes | |
| `code` | string | yes | the 6-digit emailed code, or the `reset_token` from `/auth/otp/verify` |
| `new_password` | string | yes | same rules as registration |

On success every refresh token of the account is revoked: sign in again. A
rejected weak password leaves the code or token unspent, so the user can
try another password.

200 `data`: `{ "detail": "Password reset successfully." }`.
Errors: `validation_error` (weak password in `details.new_password`),
`no_code`, `expired`, `invalid`, `attempts_exhausted`, `invalid_reset_token`,
`throttled`.

### 6.2 Me

#### GET /me

Auth: yes. 200 `data`: User.

#### PATCH /me

Auth: yes. Send only what changes. Other User fields are read-only and
ignored.

| Field | Type | Rules |
|---|---|---|
| `full_name` | string | max 150 |
| `avatar_url` | string | URL, max 500 |
| `locale` | string | max 10 |
| `timezone` | string | IANA name |

200 `data`: User. Errors: `validation_error`.

#### DELETE /me

Auth: yes. Deletes the account: frees the email, removes contacts, messages
and media, turns the inactivity switch off, withdraws trustee requests and
revokes every refresh token. Messages already delivered disappear from
recipients' inboxes. There is no undo.

200 `data`: `{ "detail": "Account deleted successfully." }`.

#### GET /me/home

Auth: yes. The home screen in one call.

```json
{
  "user": { "id": "89889aa6-d71b-4af1-a5c1-e3ae114d4233", "full_name": "Omar Ali", "...": "..." },
  "unread_notifications_count": 1,
  "has_waiting_message": true,
  "waiting_messages_count": 1,
  "recent_contacts": [],
  "scheduled_messages_count": 0
}
```

`waiting_messages_count`: received messages in the inbox not opened yet.
`scheduled_messages_count`: own messages that are `scheduled` or `paused`.
`recent_contacts`: as `GET /contacts/recent`.

#### POST /me/pin

Auth: yes. Sets the app lock PIN, or changes it.

| Field | Type | Required | Rules |
|---|---|---|---|
| `pin` | string | yes | 4 to 6 digits |
| `old_pin` | string | when a PIN is already set | the current PIN |

200 `data`: `{ "detail": "PIN saved successfully." }`.
Errors: `validation_error`, `old_pin_required`, `invalid_pin`, `throttled`.

#### POST /me/pin/verify

Auth: yes.

| Field | Type | Required |
|---|---|---|
| `pin` | string | yes |

200 `data`: `{ "valid": true }`. `valid` is `false` for a wrong PIN and when
no PIN is set. Errors: `validation_error`, `throttled`.

#### GET /me/subscription

Auth: yes.

```json
{
  "plan": {
    "id": "5aae7c92-099c-4c00-8a07-68bf1384f7ca",
    "code": "free",
    "name": "Free",
    "price": 0.0,
    "currency": "USD",
    "billing_period": "",
    "max_messages": 3,
    "storage_years": 1,
    "max_media_mb": 100,
    "features": {},
    "is_active": true
  },
  "status": "active",
  "messages_remaining": 3,
  "storage_remaining_mb": 100
}
```

`status` is always `active` (there is no billing). `messages_remaining` is
`max_messages` minus the user's confirmed (non-draft, not deleted) messages.
`storage_remaining_mb` is `max_media_mb` minus all media the user holds,
uploaded or only reserved, in whole MB.

### 6.3 Plans

#### GET /plans

Auth: no. Active plans by price, `business` last. 200 `data`: array of Plan
(as in `/me/subscription`). `code` is `free`, `monthly`, `one_time` or
`business`; `price` is a number. Seeded defaults (editable by the operator):

| code | price | max_messages | max_media_mb | storage_years |
|---|---|---|---|---|
| `free` | 0.00 | 3 | 100 | 1 |
| `monthly` | 4.99 | 50 | 2048 | 5 |
| `one_time` | 49.00 | 100 | 5120 | 25 |
| `business` | 0.00 (by arrangement) | 1000 | 51200 | 50 |

The `business` plan's `price` is `0.00` and its `billing_period` is
`custom`: the price is agreed per customer, so show "by arrangement" (or a
contact form, `POST /business-leads`) instead of the number.

### 6.4 Contacts

#### GET /contacts

Auth: yes. Sorted by name. Query: `search` (matches name, relation or
email, case-insensitive), `page_size`, `page`. 200 `data`: array of Contact.

#### POST /contacts

Auth: yes.

| Field | Type | Required | Rules |
|---|---|---|---|
| `name` | string | yes | max 150 |
| `relation` | string | no | max 60 |
| `email` | string | no | valid email |
| `phone` | string | no | max 32 |
| `is_self` | boolean | no | at most one self contact |

Example body: `{"name": "Fatima", "relation": "Mother", "email": "fatima@example.com", "is_self": false}`.

201 `data`: Contact. Errors: `validation_error`, `self_contact_exists`.

#### GET /contacts/recent

Auth: yes. Up to 10 contacts that confirmed messages went to, most recent
first. 200 `data`: array of Contact.

#### POST /contacts/import

Auth: yes. 1 to 500 items. An item whose email (case-insensitive) or phone
matches an existing contact, or an earlier item, is skipped.

```json
{
  "contacts": [
    { "name": "Layla", "phone": "+201000000002" },
    { "name": "Omar", "relation": "Brother", "email": "omar@example.com" }
  ]
}
```

Item fields as in `POST /contacts` without `is_self`; only `name` is required.

201 `data`: `{ "imported_count": 1, "contacts": [Contact] }` (only the
contacts created). Errors: `validation_error`.

#### PATCH /contacts/{id}

Auth: yes. Any of `name`, `relation`, `email`, `phone`. 200 `data`: Contact.
Errors: `validation_error`, `trustee_email_required`, `not_found`.

#### DELETE /contacts/{id}

Auth: yes. 200 `data`: `{ "detail": "Contact deleted successfully." }`.
Errors: `contact_is_trustee`, `not_found`. Messages already released to the
contact stay readable through their links.

### 6.5 Media

#### POST /media/upload-url

Auth: yes. Reserves the storage and returns where to send the bytes.

| Field | Type | Required | Rules |
|---|---|---|---|
| `kind` | string | yes | `voice`, `image`, `video` |
| `file_name` | string | yes | max 255; not used |
| `mime_type` | string | yes | `audio/*` for voice, `image/*` for image, `video/*` for video |
| `size_bytes` | integer | yes | exact file size, at least 1 |

200 `data`:

```json
{
  "media_id": "f0a5ccd9-1829-4bad-bf43-091cf564520a",
  "upload_url": "https://api.catalog.app/api/v1/media/f0a5ccd9-1829-4bad-bf43-091cf564520a/upload",
  "storage_key": "media/8f6d6c64-9eda-4ba6-9314-5208f305a825/f0a5ccd9-1829-4bad-bf43-091cf564520a"
}
```

Errors: `validation_error`, `file_too_large`, `storage_limit_reached`.

#### PUT /media/{id}/upload

Auth: yes (bearer token). Not in the RAML: this is the `upload_url`. The
body is the raw file bytes (any Content-Type); its length must equal
`size_bytes`. One upload per media.

200 `data`: MessageMedia (with its `url`).
Errors: `size_mismatch`, `already_uploaded`, `file_too_large`, `not_found`.

#### POST /media/{id}/confirm

Auth: yes. After the upload. Body optional:

| Field | Type | Required | Rules |
|---|---|---|---|
| `duration_sec` | integer | no | 0 or more |

200 `data`: MessageMedia. Errors: `validation_error`, `not_uploaded`, `not_found`.

#### DELETE /media/{id}

Auth: yes. Deletes the file and frees its storage, uploaded or not.
200 `data`: `{ "detail": "Media deleted successfully." }`.
Errors: `invalid_state` (the media is on a message that is no longer
editable), `not_found`.

#### GET /media/{id}/file?sig=...

Auth: no (the `sig` is the credential). Not in the RAML: the media `url`.
Returns the file with its own Content-Type. A wrong or expired (over 1 hour)
`sig` is `404 not_found`.

### 6.6 Messages

#### GET /messages

Auth: yes. Your own messages, newest first. Query: `status` (one status),
`is_saved` (`true`/`false`), `page_size`, `page`. 200 `data`: array of
Message.

#### POST /messages

Auth: yes. Always creates a `draft`; nothing is sent until confirm.

| Field | Type | Required | Rules |
|---|---|---|---|
| `type` | string | yes | see Message |
| `title` | string | yes | max 200 |
| `body` | string | yes | not blank |
| `recipient_ids` | string[] | yes | at least one of your contact ids |
| `trigger_type` | string | no | default `instant` |
| `deliver_at` | datetime | for `date` and `occasion` | in the future |
| `occasion_type` | string | for `occasion` | see Message |
| `occasion_note` | string | no | max 255 |
| `channel` | string | no | `in_app` (default) or `email` |
| `media_ids` | string[] | no | your uploaded media not on another message; order kept |

Example body:

```json
{
  "type": "advice",
  "title": "For your graduation day",
  "body": "I am always proud of you...",
  "recipient_ids": ["47660aef-75f9-4416-8a34-6c98e244bc6d"],
  "trigger_type": "date",
  "deliver_at": "2027-06-01T10:00:00Z",
  "channel": "email"
}
```

201 `data`: Message (status `draft`). Errors: `validation_error` (unknown
contacts or media are listed in `details.recipient_ids` / `details.media_ids`).

#### GET /messages/{id}

Auth: yes. 200 `data`: Message. Errors: `not_found`.

#### PATCH /messages/{id}

Auth: yes. Only while `draft`, `scheduled` or `paused`. Same fields as
create, all optional. `recipient_ids` and `media_ids` replace the whole
list. Switching to `instant` is allowed only on a draft. A changed
`deliver_at` (or trigger) must be in the future.

200 `data`: Message. Errors: `validation_error`, `invalid_state`, `not_found`.

#### DELETE /messages/{id}

Auth: yes. Any status. Its media files are deleted; a released message
disappears from inboxes and its links stop working.
200 `data`: `{ "detail": "Message deleted successfully." }`. Errors: `not_found`.

#### POST /messages/{id}/confirm

Auth: yes. No body. `draft` becomes `scheduled`; an `instant` message is
released at once and comes back `sent`. Counts against `max_messages`.
200 `data`: Message.
Errors: `invalid_state`, `deliver_at_passed`, `no_recipients`,
`plan_limit_reached`, `not_found`.

#### POST /messages/{id}/pause

Auth: yes. No body. `scheduled` becomes `paused`. 200 `data`: Message.
Errors: `invalid_state`, `not_found`.

#### POST /messages/{id}/resume

Auth: yes. No body. `paused` becomes `scheduled`. If its date passed while
paused it is released on the next sweep. 200 `data`: Message.
Errors: `invalid_state`, `not_found`.

#### PATCH /messages/{id}/saved

Auth: yes. Any status.

| Field | Type | Required |
|---|---|---|
| `is_saved` | boolean | yes (a real JSON boolean) |

200 `data`: Message (`is_saved_moment` updated). Errors: `validation_error`, `not_found`.

#### GET /message-types

Auth: no. Arabic labels when `Accept-Language` starts with `ar`, else
English.

```json
[
  { "key": "dua", "name": "Dua", "icon": "hands", "placeholder": "May Allah bless you and keep you safe..." },
  { "key": "advice", "name": "Advice", "icon": "lightbulb", "placeholder": "Something I learned that I want you to know..." }
]
```

(Five items: `dua`, `advice`, `love`, `apology`, `reassurance`.)

### 6.7 Inactivity

#### GET /inactivity

Auth: yes. Created with defaults on first read. The switch is brought up to
date before answering (a due check-in is sent; the state may move on).

```json
{
  "user_id": "8f6d6c64-9eda-4ba6-9314-5208f305a825",
  "enabled": true,
  "checkin_interval_days": 30,
  "missed_checkins_limit": 3,
  "grace_period_days": 7,
  "verify_by_email": true,
  "trustee_contact_id": "47660aef-75f9-4416-8a34-6c98e244bc6d",
  "state": "active",
  "next_checkin_at": "2026-11-30T00:34:17.411433Z"
}
```

`state`: `active`, `warning`, `pending_trustee`, `released`, `cancelled`.
Defaults: off, 30 days, 3 missed, 7 grace days, `verify_by_email` true, no
trustee.

#### PUT /inactivity

Auth: yes. Fields left out keep their value (it behaves like PATCH).

| Field | Type | Rules |
|---|---|---|
| `enabled` | boolean | |
| `checkin_interval_days` | integer | 1 to 365 |
| `missed_checkins_limit` | integer | 1 to 12 |
| `grace_period_days` | integer | 1 to 365 |
| `verify_by_email` | boolean | also email each check-in |
| `trustee_contact_id` | string or null | one of your contacts, with an email; `null` for none |

200 `data`: InactivitySettings. Errors: `validation_error`, `trustee_email_required`.

#### POST /checkins/respond

Auth: yes. No body. The "I'm fine" button; see 7.6.

```json
{
  "detail": "Check-in recorded. You are marked active.",
  "last_active_at": "2026-10-31T00:34:17.426212Z",
  "next_checkin_at": "2026-11-30T00:34:17.426212Z"
}
```

`next_checkin_at` is `null` when the switch is off or released.

### 6.8 Trustee

These two are the trusted contact's web page; the app itself does not call
them. The token comes from the emailed link `{frontend}/trustee/{token}`.

#### GET /trustee/{token}

Auth: no.

```json
{ "sender_name": "Sara A. Ali", "requested_at": "2026-10-31T00:34:17.435714Z", "status": "pending" }
```

`status`: `pending`, `confirmed`, `rejected` (a request the user withdrew,
by checking in or changing the trustee, also reads `rejected`).
Errors: `not_found`, `throttled`.

#### POST /trustee/{token}/decision

Auth: no.

| Field | Type | Required | Rules |
|---|---|---|---|
| `decision` | string | yes | `confirmed` or `rejected` |
| `note` | string | no | max 2000 |

200 `data`: `{ "detail": "Decision recorded successfully.", "decision": "rejected" }`.
Errors: `validation_error`, `already_decided`, `not_found`, `throttled`.

### 6.9 Inbox

#### GET /inbox

Auth: yes. Messages released to contacts linked to this account, newest
first. Empty until the account's email is verified. Query: `page_size`, `page`.

```json
[
  {
    "delivery_id": "2e6ec588-20b0-40d9-abf6-0548aad51540",
    "message_id": "3565f768-d6a0-4e7b-87e5-f60417cdd793",
    "sender_name": "Sara A. Ali",
    "sender_avatar": null,
    "title": "Just because",
    "type": "love",
    "channel": "in_app",
    "status": "delivered",
    "created_at": "2026-10-31T00:34:17.159894Z",
    "delivered_at": "2026-10-31T00:34:17.156765Z"
  }
]
```

`status`: `delivered` (in-app), `sent` (email), `opened`.

#### GET /inbox/{deliveryId}

Auth: yes. The first call marks the delivery and the message `opened` and
notifies the sender.

```json
{
  "delivery_id": "2e6ec588-20b0-40d9-abf6-0548aad51540",
  "sender_name": "Sara A. Ali",
  "type": "love",
  "title": "Just because",
  "body": "I love you, brother.",
  "media": [ { "id": "f0a5ccd9-1829-4bad-bf43-091cf564520a", "kind": "voice", "url": "https://...", "...": "..." } ],
  "sent_at": "2026-10-31T00:34:17.156765Z",
  "opened_at": "2026-10-31T00:34:17.383709Z"
}
```

Errors: `not_found`.

### 6.10 Recipient link

Public endpoints behind the link `{frontend}/r/{token}` in a delivery email.
See 7.5 for the sequence.

#### GET /r/{token}

Auth: no.

```json
{ "sender_name": "Sara A. Ali", "requires_verification": true, "title": "Read this when you need it" }
```

Errors: `not_found`, `throttled`.

#### POST /r/{token}/verify

Auth: no.

| Field | Type | Required | Rules |
|---|---|---|---|
| `email` | string | yes | the address the message was sent to |
| `code` | string | yes | the 6-digit `recipient_access` code |

200 `data`:

```json
{ "verified": true, "view_session_token": "eyJkIjoiOTA5N2I2YWQtMWZiOC00MTg0LWI0NzktOWU3YzE3YmI0NzE4In0:1xMx33:J4bz8p..." }
```

Errors: `validation_error`, `email_mismatch`, `no_code`, `expired`, `invalid`,
`attempts_exhausted`, `not_found`, `throttled`.

#### POST /r/{token}/open

Auth: no. When `requires_verification` is true, send the view session as
`{"view_session_token": "..."}` in the body or as the `X-View-Session`
header.

```json
{
  "sender_name": "Sara A. Ali",
  "title": "Read this when you need it",
  "type": "reassurance",
  "body": "Everything will be okay.",
  "media": [],
  "opened_at": "2026-10-31T00:34:17.367481Z"
}
```

The first open marks the message `opened` and notifies the sender;
`opened_at` stays the first open's time.
Errors: `verification_required`, `not_found`, `throttled`.

#### POST /r/{token}/later

Auth: no. Same session rule as open.

```json
{ "detail": "Marked to open later.", "open_later_at": "2026-10-31T00:34:17.362390Z" }
```

Errors: `verification_required`, `not_found`, `throttled`.

### 6.11 Notifications

#### GET /notifications

Auth: yes. Newest first. Query: `read` (`true`/`false`), `type` (one or
several, comma-separated), `search`, `ordering` (`created_at`, `-created_at`,
`read`, `-read`), `page_size`, `page`.

```json
[
  {
    "id": "1",
    "type": "message_received",
    "title": "New message from Sara A. Ali",
    "body": "Just because",
    "data": { "delivery_id": "2e6ec588-20b0-40d9-abf6-0548aad51540" },
    "read_at": null,
    "created_at": "2026-10-31T00:34:17.161517Z"
  }
]
```

| `type` | Sent when | `data` |
|---|---|---|
| `message_received` | a message reaches your inbox | `delivery_id` |
| `message_opened` | a recipient opens your message | `delivery_id`, `message_id` |
| `checkin` | a check-in is due | `checkin_id` |
| `trustee` | your trusted contact was asked, or could not be reached | `{}` |
| `system` | an email delivery of your message gave up for good (or an admin announcement) | `delivery_id`, `message_id` |
| `support` | a support ticket got a reply, or was resolved or closed | `{}` |

#### PATCH /notifications/{id}/read

Auth: yes. No body.

```json
{ "id": "1", "read_at": "2026-10-31T00:34:17.454795Z" }
```

Already-read notifications keep their first `read_at`. Errors: `not_found`.

Also available, outside the RAML: `GET /notifications/unread-count`
(`{"unread_count": 2}`), `POST /notifications/mark-all-read`
(`{"updated_count": 2}`), `DELETE /notifications/clear-all` (deletes read
ones, `{"cleared_count": 2}`), and `GET/PATCH/DELETE /notifications/{id}`.

### 6.12 Devices

#### POST /devices

Auth: yes. Registering the same token again updates it (and moves it to
the current account).

| Field | Type | Required | Rules |
|---|---|---|---|
| `fcm_token` | string | yes | max 512 |
| `platform` | string | yes | `ios`, `android`, `web` |

201 `data`: `{ "detail": "Device registered successfully." }`. Errors: `validation_error`.

### 6.13 Support

#### POST /business-leads

Auth: no.

| Field | Type | Required | Rules |
|---|---|---|---|
| `name` | string | yes | max 255 |
| `company` | string | yes | max 255 |
| `email` | string | yes | valid email |
| `phone` | string | no | max 50 |
| `message` | string | yes | |

201 `data`: `{ "detail": "Lead submitted successfully." }`.
Errors: `validation_error`, `throttled`.

#### GET /faq

Auth: no. Published questions in display order.

```json
[ { "question": "Who can read my messages?", "answer": "Only the people you choose.", "category": "Privacy" } ]
```

`category` is the category name or `null`.

Outside the RAML, the support app also serves `/legal` and `/legal/{slug}`
(published terms and policies) and the ticket endpoints under `/support/`
(see Swagger, tag Support). Ticket writes need a verified email
(`email_not_verified`).

### 6.14 System

`GET /health` (no auth): `{"status": "ok", "database": "ok"}`, or `503` with
code `error` when the database is down.

`GET /force-update?force_update_code=5` (no auth) says whether this build must
update. `force_update_code` is the integer build code from the app's pubspec
YAML, 1 or higher. Otherwise the response is `400 validation_error`. The server
compares the code with the minimum code for the `X-Platform` platform, set in
the Django admin.

A code below the minimum:

```json
{
  "force_update": true,
  "title": "Update required",
  "body": "Please update to continue.",
  "link": "https://apps.apple.com/app/id1"
}
```

A code at or above the minimum, or a platform with no active minimum:

```json
{
  "force_update": false,
  "title": null,
  "body": null,
  "link": null
}
```

Endpoints under `/admin/...`, `/support/admin/...` and
`/notifications/admin/send` belong to the admin console and need RBAC
permissions. They are listed in Swagger under the Admin tag.

## 7. Flows

### 7.1 Sign up and verify the email

1. `POST /auth/register` → store `access` and `refresh`. A `verify_email`
   code is emailed.
2. Show the code screen. `POST /auth/otp/verify`
   `{"target": email, "purpose": "verify_email", "code": "123456"}` →
   `{"verified": true}`.
3. To resend: `POST /auth/otp/send` `{"target": email, "purpose": "verify_email"}`
   (at most once a minute).
4. `GET /me`: `email_verified_at` is now set.

A code is valid for 5 minutes and allows 5 wrong guesses. Both are server
defaults the operator can change (`VERIFICATION_CODE_TTL_MINUTES`,
`VERIFICATION_CODE_MAX_ATTEMPTS`), so read the `invalid` message for the
attempts left rather than counting in the app. The same applies to the
reset token's 10 minutes in 7.2 (`PASSWORD_RESET_TOKEN_TTL_MINUTES`).

Until step 2, the user can use the app (contacts, messages), but nothing
reaches their in-app inbox: other people's contacts are linked to the
account only once its email is verified.

### 7.2 Password reset

1. `POST /auth/password/forgot` `{"email": ...}` → a 6-digit code is emailed.
2. Either:
   - send the code directly: `POST /auth/password/reset`
     `{"email": ..., "code": "123456", "new_password": ...}`; or
   - check it first (separate screen): `POST /auth/otp/verify`
     `{"target": email, "purpose": "reset_password", "code": "123456"}` →
     `reset_token`, then `POST /auth/password/reset`
     `{"email": ..., "code": "<reset_token>", "new_password": ...}` within
     10 minutes.
3. Every old session is revoked. `POST /auth/login` with the new password.

### 7.3 Create a message with media

1. For each file: `POST /media/upload-url`
   `{"kind": "voice", "file_name": "hello.m4a", "mime_type": "audio/mp4", "size_bytes": 48213}`
   → `media_id`, `upload_url`.
2. `PUT <upload_url>` with the bearer token and the raw bytes as the body
   (`Content-Length` = `size_bytes`).
3. `POST /media/{media_id}/confirm` (optionally `{"duration_sec": 12}`).
4. `POST /messages` with `recipient_ids` and `media_ids` → a `draft`.
5. Review screen: `GET /messages/{id}`, then `POST /messages/{id}/confirm`.

Media that never ends up on a message still counts against storage until
`DELETE /media/{id}`. Delete abandoned uploads.

Life cycle:

```
          PATCH (edit)          PATCH (edit)
             ┌──┐                  ┌──┐
             ▼  │                  ▼  │
POST ──▶  draft ──confirm──▶  scheduled ──release──▶ sent ──first open──▶ opened
                  │             ▲    │
                  │      resume │    │ pause
                  │             │    ▼
                  │            paused  (PATCH allowed)
                  │
                  └── instant: released at confirm, comes back as sent
```

- Edit (`PATCH`) only in `draft`, `scheduled`, `paused`.
- `confirm` only from `draft`; `pause` only from `scheduled`; `resume` only
  from `paused`. Anything else is `invalid_state`.
- `DELETE` works in any status.
- `opened` means at least one recipient opened it.

### 7.4 Delivery

When a message is released:

| trigger | released |
|---|---|
| `instant` | at `confirm` |
| `date`, `occasion` | when `deliver_at` passes |
| `inactivity` | when the user's inactivity switch releases (7.6) |

Release creates one delivery per recipient contact (deleted contacts are
skipped):

- `in_app`: the delivery is `delivered` at once. If the contact is linked to
  a Catalog account, it appears in that account's `GET /inbox` with a
  `message_received` notification. A contact that is not linked never sees
  an in-app message.
- `email`: an email with the link `{frontend}/r/{token}` goes to the
  contact's address. A contact without an email cannot receive it; the
  sender gets a `system` notification. Failed sends are retried (up to 5
  times, 10 minutes apart and growing) before the sender is told.

Timing: a server job (`process_due`) runs every 5 minutes. It releases due
messages and sends queued email. Requests that show messages (`/inbox`,
`/me/home`, `/messages`, `/r/{token}`) also release anything due, so in-app
messages appear as soon as someone looks. Expect:

- `instant` email: sent right after confirm.
- `date`/`occasion` in-app: visible from `deliver_at`, at the next request
  or job run.
- `date`/`occasion` email: within about 5 minutes after `deliver_at`.

Accounts that are suspended, deactivated by an admin or deleted release
nothing scheduled, dated or inactivity. An admin deactivation keeps the
account's messages and media and can be undone; the inactivity switch
stays off until the user turns it on again.

### 7.5 Recipient link

For someone who received an email delivery (the app can host this screen
for a deep link, or the web frontend does):

1. `GET /r/{token}` → `sender_name`, `title`, `requires_verification`.
2. If `requires_verification` is false: go to step 5.
3. `POST /auth/otp/send`
   `{"target": recipient_email, "purpose": "recipient_access"}` → a code is
   emailed to that address.
4. `POST /r/{token}/verify` `{"email": recipient_email, "code": "123456"}` →
   `view_session_token`, valid 30 minutes for this link only.
5. `POST /r/{token}/open` with `{"view_session_token": "..."}` in the body
   or `X-View-Session: ...` → the message. Or `POST /r/{token}/later` to save
   it for later (it stays unopened).

After 30 minutes, repeat steps 3 and 4.

### 7.6 Inactivity switch

1. Choose a trusted contact (optional; it needs an email) and settings:
   `PUT /inactivity` `{"enabled": true, "checkin_interval_days": 30, "missed_checkins_limit": 3, "grace_period_days": 7, "verify_by_email": true, "trustee_contact_id": "..."}`.
   Turning it on starts a fresh interval: `next_checkin_at` = now + interval.
2. At `next_checkin_at` a check-in is sent: a `checkin` notification, plus an
   email when `verify_by_email` is true. State becomes `warning`, and the
   next check-in is one interval later.
3. A check-in counts as **missed** once it has gone a full interval
   unanswered (sent at T, missed from T + interval).
4. When missed check-ins reach `missed_checkins_limit`:
   - with a trusted contact: state `pending_trustee`, and the contact is
     emailed a link to the trustee page. Only their `confirmed` releases;
     time alone never does.
   - without one: messages are released once `grace_period_days` have
     passed since the limit was reached.
5. Release: state `released`, every `scheduled` `inactivity` message is
   released, and the account's `status` becomes `deceased_confirmed`. This
   cannot be undone.

`POST /checkins/respond` ("I'm fine") at any point before release: answers
every open check-in, sets `last_active_at`, withdraws a pending trustee
request, and restarts the interval (state `active`, `next_checkin_at` = now +
interval). Logging in does **not** count as a check-in.

Other changes:

- `enabled: false` → state `cancelled`, pending requests withdrawn.
- Changing the trusted contact while `pending_trustee` withdraws the old
  request and asks the new contact (or, with no contact, starts the grace
  period).
- Trustee `rejected` → state `active` with a fresh interval.
- The trusted contact cannot be deleted or lose its email
  (`contact_is_trustee`, `trustee_email_required`).

### 7.7 Plans and limits

- `max_messages`: confirmed messages (any status but `draft`, not deleted).
  Checked at `POST /messages/{id}/confirm` → `plan_limit_reached`. Drafts are
  unlimited.
- `max_media_mb`: total size of all the user's media, uploaded or only
  reserved by `upload-url`. Checked at `POST /media/upload-url` →
  `storage_limit_reached`.
- Per-file cap: 100 MB by default (server setting), at upload-url and at
  the upload → `file_too_large`.
- `GET /me/subscription` shows what is left.

## 8. Differences from RAML

The code is what this guide and Swagger describe. Where it differs from
`RAML/RAML.yaml`:

1. **Extra endpoints.** `PUT /media/{id}/upload` (raw bytes, the
   `upload_url`), `GET /media/{id}/file?sig=` (the signed media `url`) and
   `GET /force-update` exist only in the code. The app uses the URLs it is
   given. The required `X-Platform` header is also not in the RAML.
2. **`/auth/refresh` errors.** An expired, malformed or revoked refresh
   token is `401 unauthorized`, not 400. A missing `refresh` field is
   `400 validation_error`.
3. **One 401 code.** Every authentication failure uses
   `error.code: "unauthorized"`; the message differs.
4. **Extra 400 codes.** Beyond `validation_error`, endpoints return the
   specific codes in section 4 (`contact_is_trustee`,
   `trustee_email_required`, `self_contact_exists`, `plan_limit_reached`,
   `invalid_state`, OTP codes, and so on).
5. **`/notifications` is GET only.** POST answers 405. Extra notification
   endpoints exist (6.11). Notification ids are integers sent as strings.
6. **Optional fields with defaults.** `POST /messages`: `trigger_type`
   (default `instant`) and `channel` (default `in_app`) are optional.
   `POST /contacts`: `relation` is optional. `PUT /inactivity`: every field
   is optional (partial update).
7. **Paging.** List endpoints accept the opt-in `page_size`/`page` query
   (bare array otherwise).
8. **More statuses than listed.** Endpoints with an id in the path answer
   `404 not_found` for ids that do not exist or are not the caller's, also
   where the RAML lists no 404 (for example `DELETE /contacts/{id}`,
   `POST /messages/{id}/pause`). `POST /messages/{id}/pause` and `/resume`
   answer `400 invalid_state`, `PATCH /messages/{id}/saved` answers
   `400 validation_error`, and `DELETE /contacts/{id}` answers
   `400 contact_is_trustee`, none of which the RAML lists. Throttled
   endpoints answer `429` (section 5).
9. **Extra list filters.** `/notifications` takes `read`, `type`, `search`,
   `ordering`.
10. **Empty strings, not missing keys.** Optional string fields are always
    present, as `""` when empty (`phone`, `avatar_url`, `relation`,
    `occasion_note`); `occasion_type` is `null` when unset.
11. **`/health`** and the `/legal`, `/support/...` and admin endpoints are
    not in the RAML.

## 9. Changelog

- v1 — 2026-10-31 — initial.
