# Katalog Admin Dashboard — Design

Date: 2026-10-31

## Intent

One place for the Katalog team to run the whole app: see what is happening,
manage users and plans, handle support, and fix delivery problems — without
opening the database or Django admin. User asked for it "to handle every thing
in this app", built in **Flutter**, in a **new folder**, implemented without
further questions, styled after the Behance project "Catalog App — Product
Design — Future Memory Vault".

## Decisions (builder's calls; user said "implement without asking")

### Platform and location
- Flutter **web** app (desktop browser first, responsive down to tablet width).
  Same code can build for macOS/Windows later.
- Folder: `/Users/abdalrhmanreda/Codex/Backend/Django/katalog_dashboard/`,
  its own git repo, sibling of the backend `Catalog/`.
- Talks only to the Katalog API (`/api/v1/`), same envelope and JWT as Android.
- Packages: `dio` (HTTP), `flutter_riverpod` (state), `go_router` (routing),
  `fl_chart` (charts), `google_fonts`, `intl`, `shared_preferences`
  (refresh token only). Tests: `flutter_test`, `mocktail`.
- API base URL from `--dart-define=API_BASE_URL=...`, default
  `http://localhost:8000/api/v1`.

### Who signs in
- Staff accounts only. Access is the backend's RBAC: the dashboard shows a
  section only if the signed-in user holds its `*.read` permission, and shows
  an action only if they hold the matching write permission. The backend
  enforces the same rules; the UI hiding is convenience, not security.
- Login: `POST /auth/login`, then `GET /admin/control/me` (new) for the
  permission list. A user with no `dashboard.read` gets "No dashboard access".
- Access token in memory; refresh token in `shared_preferences`; on 401
  refresh once, then sign out.

### Privacy line (hard rule)
- The dashboard never shows message bodies, media files, media URLs, PINs,
  OTP codes, or any token. Messages appear as metadata only: id, sender,
  type, title, status, trigger, channel, deliver_at, sent_at, recipient
  count, delivery counts. The backend admin endpoints never return these
  fields, so a compromised dashboard can't leak them either.
- No admin action can release an inactivity message, confirm a trustee
  decision, or change a switch state to `released`. Inactivity is read-only
  in the dashboard except "cancel switch" is NOT offered either.

### Sections
1. **Overview** — KPI cards (users: total, new 7d, verified %, suspended;
   messages by status; deliveries sent/failed/queued 24h; switches by state;
   pending trustee requests; open tickets; new leads) and two charts
   (sign-ups per day, deliveries per day, last 30 days).
2. **Users** — searchable, filterable table (status, verified, plan);
   detail page: profile, plan, counts (contacts, messages by status, devices),
   inactivity state, roles. Actions: suspend / unsuspend, deactivate /
   restore (existing admin soft-delete, reversible), change plan, mark email
   verified, assign roles.
3. **Messages** — metadata table filtered by status, trigger, channel, sender;
   detail shows deliveries per recipient (contact name masked to first
   letter + relation, channel, status, attempts, last_error, sent/opened).
4. **Deliveries** — table of deliveries filtered by status (failed/queued
   first), action **retry** (calls the same resend path `process_due` uses;
   fresh token, never shown).
5. **Inactivity** — switches table filtered by state; pending trustee
   requests with requested_at and email_sent_at. Read-only.
6. **Plans** — CRUD for plans (name, price, currency, billing period, limits,
   features JSON, active). Deleting a plan in use is refused.
7. **Support** — existing template admin endpoints: tickets (reply, status),
   FAQ CRUD, categories, business leads (status).
8. **Notifications** — broadcast (existing `notifications/admin/send`).
9. **Legal** — legal documents editor (existing admin legal endpoints).
10. **Access** — roles and permissions (existing rbac endpoints).
11. **System** — system logs, audit log, deployments, health.

### Backend additions (in `Catalog/`, under `/api/v1/admin/control/`)
- `GET me` — current staff user + permission slugs.
- `GET stats/overview` and `GET stats/timeseries?metric=signups|deliveries&days=30`.
- Users: extend existing admin user endpoints with filters (status,
  verified, plan), detail counts, `POST users/{id}/suspend`,
  `POST users/{id}/unsuspend`, `POST users/{id}/verify-email`,
  `PATCH users/{id}/plan`.
- `GET messages`, `GET messages/{id}` (metadata + deliveries).
- `GET deliveries`, `POST deliveries/{id}/retry`.
- `GET inactivity`, `GET inactivity/trustee-requests`.
- `GET/POST plans`, `PATCH/DELETE plans/{id}`.
- RBAC catalog adds: `messages.read`, `deliveries.read`, `deliveries.retry`,
  `inactivity.read`, `plans.read|create|update|delete`, `users.suspend`;
  default roles updated (admin/manager/support/viewer as in template spirit).
- Every write is audit-recorded (`rbac.audit.record`).
- All list endpoints paginate with `?page=&page_size=` (default 25, max 200)
  and return `{count, next, previous, results}` inside `data`.
- CORS: dashboard origin configurable via existing `DJANGO_CORS_ALLOWED_ORIGINS`.

### Visual style
- Behance page could not be fetched (HTTP 403), so the direction is built from
  the project name "Future Memory Vault": calm, intimate, archival.
  Dark-first: deep midnight ink background, warm parchment text, soft gold
  accent for primary actions, muted status colors (sage = ok, amber = warning,
  clay = error, mist = neutral). Serif display face for headings (e.g.
  Fraunces), clean sans for data (e.g. Inter). Rounded 16px cards, generous
  spacing, quiet dividers, no heavy shadows.
- All tokens live in `lib/theme/tokens.dart`; replacing the palette/fonts
  with the exact Behance values is a one-file edit. Light theme provided too.
- Arabic: UI strings in English first, structured for `intl` so Arabic/RTL
  can be added; layout uses directional padding.

## Goals
- Every section above working end-to-end against the real API.
- Staff with partial permissions see only what they may use.
- Tests: backend endpoint tests (permission 403, privacy fields absent,
  filters, audit); Flutter unit tests for API client/auth/refresh and widget
  tests per section.

## Non-goals
- Reading message content or media; impersonating users.
- Editing inactivity state or trustee decisions.
- Payments (none in this product).
- Push notification sending.
- Mobile phone layout (tablet and up only).

## Anti-goals
- Any endpoint or screen that exposes a message body, media, token, OTP, PIN.
- Admin action that releases messages or marks a user deceased.
- UI that relies on hiding buttons for security.

## Left to the builder
Exact layout of each page, chart styling, empty states, copy, icon choice.
