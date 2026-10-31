# Katalog Admin Dashboard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A Flutter web admin dashboard (`katalog_dashboard/`) that runs every part of Katalog through new and existing admin API endpoints in the Django backend.

**Architecture:** Backend first: add metadata-only admin endpoints under `/api/v1/admin/control/` guarded by `rbac.HasPermission`. Then a Flutter web app: one API client that unwraps the envelope, an auth controller with refresh, a permission-aware shell (sidebar + router), and one feature folder per section.

**Tech Stack:** Django 6 / DRF (backend, existing). Flutter 3.44 / Dart 3.12, `dio`, `flutter_riverpod`, `go_router`, `fl_chart`, `google_fonts`, `intl`, `shared_preferences`, `mocktail`.

**Spec:** `docs/superpowers/specs/2026-10-31-katalog-dashboard-design.md` (in the Catalog repo). Backend contract for the Android app: `RAML/RAML.yaml` (must stay unchanged). Admin endpoints are new and documented in OpenAPI only.

## Global Constraints

- Backend: response envelope `{"status_code","data","error"}`; routes under `/api/v1/` via `core.routing.route` (optional trailing slash); ids strings.
- Admin endpoints: `permission_classes = [HasPermission]` with `required_permission(s)`; never check role names or `is_staff`. Writes call `rbac.audit.record(...)` inside the transaction.
- Admin list endpoints: paginated, `?page=&page_size=` default 25 max 200, `data = {count, next, previous, results}`.
- Privacy: admin responses never include message body/ciphertext/iv/tag, media `storage_key`/`url`, tokens or token hashes, OTP codes, PIN hashes, passwords. Tests assert these keys are absent.
- No admin endpoint changes inactivity state to `released`, decides trustee requests, or releases messages.
- RAML endpoints, their tests and `docs/openapi.yaml` drift test stay green; regenerate `docs/openapi.yaml` after adding admin views.
- Backend gate: `venv/bin/ruff check . && venv/bin/mypy . && venv/bin/python manage.py check && venv/bin/python manage.py makemigrations --check --dry-run && venv/bin/python manage.py spectacular --file /tmp/openapi.yaml --validate --fail-on-warn && venv/bin/python manage.py test --noinput` (suite ~5 min: redirect to a file, long timeout).
- Flutter: app in `/Users/abdalrhmanreda/Codex/Backend/Django/katalog_dashboard/`, own git repo (`main` branch). Web target. `flutter analyze` clean (default `flutter_lints`), `flutter test` green, `flutter build web` succeeds.
- Flutter: all colors, fonts, radii, spacing come from `lib/theme/tokens.dart` — no literal colors elsewhere. Dark theme default, light theme available.
- Flutter: UI hides what the user lacks permission for, but every screen still handles a 403 from the API gracefully ("You don't have access to this").
- Commits: Conventional Commits, each message ending with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. No push.

## Review Focus

1. Staff user without `messages.read` calls `GET /admin/control/messages` → 403 envelope; dashboard shows the no-access state, not a crash. Owners: B3, F4.
2. A message with media and a body: admin message detail JSON contains none of `body`, `body_encrypted`, `storage_key`, `url`, `access_token_hash`. Owner: B3.
3. Access token expires mid-session: one refresh, request replayed; refresh fails → login screen. Owner: F1.
4. Deleting a plan that users are on → 400 `plan_in_use`. Owner: B4.
5. Suspending a user blocks their next API call (existing `ActiveUserJWTAuthentication`) and is audit-logged. Owner: B2.

---

## Backend (repo `Catalog/`, branch `feat/katalog-dashboard-api` cut from `feat/katalog-backend`)

### Task B1: Admin permissions, `admin/control/me`, overview stats and time series

**Files:** Modify `rbac/catalog.py`, `rbac/views.py` or new `dashboard_api/` app? → Create app `adminapi/` (models-free) with `views/`, `serializers.py`, `urls.py`, `tests/`; mount at `api/v1/` in `config/urls.py`.

**Interfaces:**
- Produces permissions in `rbac/catalog.py`: resources `messages`, `deliveries`, `inactivity`, `plans`; slugs `messages.read`, `deliveries.read`, `deliveries.retry`, `inactivity.read`, `plans.read`, `plans.create`, `plans.update`, `plans.delete`, `users.suspend`. Default roles: admin = all except `system_logs.delete` (as today); manager adds `messages.read`, `deliveries.read`, `deliveries.retry`, `inactivity.read`, `plans.read`, `users.suspend`; support adds `messages.read`, `deliveries.read`, `inactivity.read`; viewer adds `messages.read`, `deliveries.read`, `inactivity.read`, `plans.read`. A data migration (or `seed_rbac`) syncs them.
- Produces `GET /api/v1/admin/control/me` → `{"user": UserSerializer, "permissions": [slug...], "roles": [{"slug","name"}]}`. Requires authentication only (any staff or not; returns empty permissions for non-staff). If an equivalent `admin/me` exists in rbac, reuse its logic and alias the new path.
- Produces `GET /api/v1/admin/control/stats/overview` (`dashboard.read`) → `{"users": {"total","new_7d","verified","suspended","deleted"}, "messages": {<status>: n for all 7 statuses}, "deliveries_24h": {"sent","delivered","opened","failed","queued"}, "switches": {<state>: n for 5 states, "enabled": n}, "trustee_pending": n, "tickets_open": n, "leads_new": n}`.
- Produces `GET /api/v1/admin/control/stats/timeseries?metric=signups|deliveries&days=30` (`dashboard.read`, days 1–365) → `{"metric","days","points":[{"date":"YYYY-MM-DD","value":n}]}` with zero-filled days, UTC.

- [ ] Step 1: failing tests `adminapi/tests/test_stats.py`: permission 403 without `dashboard.read`; overview keys exact; counts correct for seeded data; timeseries zero-fill and bad metric → 400 `validation_error`; `me` returns permission slugs for a role holder.
- [ ] Step 2: run, see FAIL.
- [ ] Step 3: implement.
- [ ] Step 4: backend gate green; regenerate `docs/openapi.yaml`.
- [ ] Step 5: commit.

### Task B2: Admin users — filters, detail, suspend/unsuspend, verify email, change plan

**Files:** Modify `accounts/admin_views.py`, `accounts/urls.py` (or move into `adminapi/`), tests `adminapi/tests/test_users.py`.

**Interfaces:**
- `GET /admin/control/users?search=&status=active|suspended|deceased_confirmed&verified=true|false&plan=<code>&deleted=true|false&page=&page_size=` (`users.read`) → paginated `AdminUserSerializer`: `id, full_name, email, phone, status, is_active, email_verified_at, plan_code, last_active_at, created_at, deleted_at, roles:[slug]`. Keep existing template behavior working; extend, don't break.
- `GET /admin/control/users/<uuid:pk>` (`users.read`) → `AdminUserSerializer` + `counts: {contacts, devices, messages: {<status>: n}}`, `inactivity: {enabled, state, next_checkin_at}` or null.
- `POST .../suspend` and `.../unsuspend` (`users.suspend`) → user; suspend blacklists refresh tokens; cannot suspend yourself (400 `cannot_suspend_self`); audit `USER_SUSPENDED`/`USER_UNSUSPENDED` (add `AuditAction` values + migration).
- `POST .../verify-email` (`users.update`) → sets `email_verified_at` if null and sends `email_status_changed` (so contacts link, as C1 does); audit `USER_UPDATED`.
- `PATCH .../plan` `{"plan": "<code>"}` (`users.update`) → user; unknown/inactive plan → 400 `validation_error`; audit.

- [ ] Steps 1–5 as B1 (tests: each permission, self-suspend refused, suspended user's access token rejected on `/me`, verify-email links contacts, audit rows exist, filters).

### Task B3: Admin messages and deliveries (metadata only), delivery retry

**Files:** Create `adminapi/views/messages.py`, serializers, tests `adminapi/tests/test_messages.py`.

**Interfaces:**
- `GET /admin/control/messages?status=&trigger_type=&channel=&sender=<uuid>&search=<title>&page=` (`messages.read`) → paginated `{id, sender:{id,full_name,email}, type, title, status, trigger_type, channel, deliver_at, sent_at, created_at, deleted_at, recipient_count, deliveries:{<status>:n}}`.
- `GET /admin/control/messages/<uuid:pk>` (`messages.read`) → same + `media_count`, `deliveries:[{id, contact:{id, name_initial, relation}, channel, status, attempts, last_error, sent_at, opened_at, open_later_at}]`. `name_initial` = first letter of contact name + "." .
- `GET /admin/control/deliveries?status=&channel=&message=<uuid>&page=` (`deliveries.read`) → paginated delivery rows as above + `message:{id,title,sender_email}`; default order: failed and queued first, then newest.
- `POST /admin/control/deliveries/<uuid:pk>/retry` (`deliveries.retry`) → delivery after retry. Only `failed` or `queued` email deliveries on non-deleted messages (else 400 `invalid_state`); reuse `messaging.services` resend path (fresh token, never returned); audit `DELIVERY_RETRIED`.
- Privacy test: create a message with body + uploaded media + delivery; assert serialized JSON of every endpoint lacks keys `body`, `body_encrypted`, `body_iv`, `body_tag`, `storage_key`, `url`, `access_token_hash`, and lacks the plaintext body string and the raw token.

- [ ] Steps 1–5.

### Task B4: Admin inactivity (read-only) and plans CRUD

**Files:** Create `adminapi/views/inactivity.py`, `adminapi/views/plans.py`, tests.

**Interfaces:**
- `GET /admin/control/inactivity?state=&enabled=&page=` (`inactivity.read`) → `{user:{id,full_name,email,status}, enabled, state, checkin_interval_days, missed_checkins_limit, grace_period_days, has_trustee, next_checkin_at, limit_reached_at, updated_at}`.
- `GET /admin/control/inactivity/trustee-requests?decision=pending|confirmed|rejected&page=` (`inactivity.read`) → `{id, user:{id,full_name,email}, trustee:{name_initial, relation}, requested_at, decision, decided_at, email_sent_at, email_attempts}`. No token/hash, no IP.
- No write endpoints for inactivity (test: POST/PATCH → 405).
- `GET/POST /admin/control/plans` (`plans.read` / `plans.create`), `GET/PATCH/DELETE /admin/control/plans/<uuid:pk>` (`plans.read|update|delete`): all Plan fields + `users_count`. `code` unique, choices from model; DELETE when `users_count > 0` → 400 `plan_in_use`; audit `PLAN_CREATED|UPDATED|DELETED` (re-add those AuditAction values).

- [ ] Steps 1–5.

---

## Flutter dashboard (repo `katalog_dashboard/`)

### Task F1: Scaffold, theme, API client, auth, shell

**Files:** `flutter create --platforms web --org app.katalog katalog_dashboard`; `lib/main.dart`, `lib/app.dart`, `lib/theme/tokens.dart`, `lib/theme/theme.dart`, `lib/core/api/api_client.dart`, `lib/core/api/api_error.dart`, `lib/core/api/page.dart`, `lib/core/auth/auth_controller.dart`, `lib/core/auth/token_store.dart`, `lib/core/router.dart`, `lib/shell/app_shell.dart` (sidebar + top bar), `lib/features/login/login_page.dart`, `lib/widgets/` (KpiCard, StatusChip, DataTablePage with pagination/search/filters, EmptyState, ErrorState, NoAccessState, ConfirmDialog), `test/...`, `README.md`, `.gitignore`.

**Interfaces:**
- `ApiClient` (dio): `Future<T> get<T>(path, {query, parse})`, `post`, `patch`, `put`, `delete`; unwraps envelope → `data`; non-2xx → throws `ApiError(statusCode, code, message, details)`; base URL from `const String.fromEnvironment('API_BASE_URL', defaultValue: 'http://localhost:8000/api/v1')`; bearer header from `AuthController`; on 401 (`code == "unauthorized"`) refreshes once via `POST /auth/refresh` and replays; second 401 → `AuthController.signOut()`.
- `Paged<T>`: `count, next, previous, results`.
- `AuthController` (Riverpod `Notifier<AuthState>`): `signIn(email, password)`, `signOut()`, `restore()` (refresh token from `shared_preferences`), `bool can(String permission)`; loads `GET /admin/control/me`; no `dashboard.read` → state `noAccess`.
- Router: `/login`, `/` (overview), `/users`, `/users/:id`, `/messages`, `/messages/:id`, `/deliveries`, `/inactivity`, `/plans`, `/support/tickets`, `/support/tickets/:id`, `/support/faqs`, `/support/categories`, `/support/leads`, `/notifications`, `/legal`, `/access/roles`, `/system/logs`, `/system/audit`, `/system/deployments`; redirect to `/login` when signed out; sidebar entries filtered by `can(<section>.read)`.
- Tokens: `KColors` (ink 950…50, parchment, gold accent, sage/amber/clay/mist status), `KType` (Fraunces display, Inter body via google_fonts), `KRadius` (8/12/16/24), `KSpace` (4-pt scale), plus `ThemeData buildDarkTheme()` / `buildLightTheme()`.

- [ ] Step 1: tests — `api_client_test.dart` (envelope unwrap, ApiError mapping, 401 → refresh → replay, refresh fail → signOut) with a mocked dio adapter; `auth_controller_test.dart` (signIn loads permissions, noAccess); `shell_test.dart` (sidebar hides sections without permission).
- [ ] Step 2: run, FAIL. Step 3: implement. Step 4: `flutter analyze && flutter test && flutter build web` green. Step 5: `git init`, commit.

### Task F2: Overview
KPI grid from `stats/overview`, two `fl_chart` line charts from `stats/timeseries` (signups, deliveries; 7/30/90-day toggle), quick links (failed deliveries, pending trustee, open tickets, new leads). Widget tests with a fake ApiClient (renders numbers; error state; loading state).

### Task F3: Users
List page (search, status/verified/plan filters, pagination), detail page (profile, counts, inactivity, roles), actions with confirm dialogs: suspend/unsuspend, deactivate/restore, verify email, change plan, edit roles (`admin/control/users/<id>/roles`). Actions shown only with the needed permission. Widget tests.

### Task F4: Messages and deliveries
Messages list + detail (metadata, deliveries table), deliveries list (status filter, failed first) with Retry action. A visible note on message detail: "Message content is private and never shown here." Widget tests incl. 403 → NoAccessState.

### Task F5: Inactivity and plans
Inactivity switches table + trustee requests tab (read-only). Plans table + create/edit form (validates numbers, features JSON editor as key/value rows) + delete with `plan_in_use` error shown. Widget tests.

### Task F6: Support
Tickets list (status/priority filters) + ticket thread with reply box and status change; FAQs CRUD; categories CRUD; leads list with status change. Uses existing `/support/admin/...` endpoints. Widget tests.

### Task F7: Notifications, legal, access, system
Broadcast form (`notifications/admin/send`); legal documents list/editor (`admin/legal`); roles & permissions matrix (`admin/control/roles`, `permissions`); system logs, audit log (filters), deployments, health indicator in top bar. Widget tests.

### Task F8: Finish
README (run: `flutter run -d chrome --dart-define=API_BASE_URL=...`; build: `flutter build web --release --dart-define=...`; CORS note: add the dashboard origin to backend `DJANGO_CORS_ALLOWED_ORIGINS`; create a staff user: `createsuperuser` + `seed_rbac`). Smoke run: backend `runserver` + `flutter build web` served locally; sign in as superuser, open every section, confirm no errors in API responses (curl the same endpoints). Report results.
