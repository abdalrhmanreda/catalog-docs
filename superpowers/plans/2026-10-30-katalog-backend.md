# Katalog Backend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A Django backend at `Catalog/` that serves every endpoint in `RAML/RAML.yaml` under `/api/v1/`, built from the backend-setup template.

**Architecture:** Copy the template into the repo root, strip social sign-in and billing, swap in a UUID email-login `User`, then add domain apps `plans`, `contacts`, `messaging`, `inactivity`. Response envelope and routing are fixed once in `core`. Time-based work runs in services called both at request time and by a `process_due` command.

**Tech Stack:** Python 3.12+, Django 6.0, DRF 3.18, SimpleJWT 5.5, PostgreSQL (psycopg 3), `cryptography` (AES-GCM), drf-spectacular.

**Spec:** `docs/superpowers/specs/2026-10-30-katalog-backend-design.md`. Contract: `RAML/RAML.yaml`. Data model: `docs/DATABASE_ERD.md`. Executors read all three.

## Global Constraints

- Every response body: `{"status_code": <int>, "data": <payload or null>, "error": null | {"message": str, "code": str, "details"?: obj}}`. Key is `error`, not `errors`.
- Every route lives under `/api/v1/` and matches with and without a trailing slash.
- Field names and enum values copied verbatim from the RAML. Ids are strings in JSON.
- `SIMPLE_JWT["ROTATE_REFRESH_TOKENS"] = False`; `AUTH_REFRESH_IN_BODY = True`.
- No Google/Microsoft sign-in, no `billing` app, no `google-auth` dependency.
- `MESSAGE_ENCRYPTION_KEY` (base64 of 32 bytes) required when `DJANGO_DEBUG` is off.
- Owner-scoped endpoints return 404 (not 403) for another user's object.
- Template rules still hold: settings from env, `rbac.HasPermission` for admin endpoints, `rbac.audit.record(...)` for security-sensitive changes.
- Test runner: `venv/bin/python manage.py test`. Database: local PostgreSQL, `POSTGRES_DB=katalog`.
- Git: repo on `main`, remote `origin` = `https://github.com/abdalrhmanreda/catalog-backend.git`. Each task ends with a full test run, then one commit (Conventional Commits, e.g. `feat(contacts): ...`). Keep `.env`, `venv/`, `media/`, `private_media/` out of git (template `.gitignore`). Push only when the user asks.

## Review Focus

1. Android sends paths without a trailing slash (`/api/v1/auth/login`) as POST. Expect 200, not a 301/404. Test owner: Task 2.
2. A refresh token used twice (Android stores it once). Expect both refreshes to succeed. Test owner: Task 4.
3. User A requests user B's message, contact, media, delivery or notification by id. Expect 404 with the envelope. Test owners: Tasks 6, 7, 8, 9.
4. A `date` message whose `deliver_at` passed while no timer ran. Expect it in the recipient's `/inbox` on the next request. Test owner: Task 9.
5. Trustee set, check-ins missed, trustee silent past the grace period. Expect messages **not** released. Test owner: Task 10.

---

### Task 1: Bootstrap from template, strip social sign-in and billing

**Files:**
- Copy: every tracked file of `https://github.com/abdalrhmanreda/backend-setup` (except `.git/`, `docs/superpowers/`) into `Catalog/`. `Catalog/requirements.txt` is replaced by the template's.
- Delete: `billing/`, `accounts/google_auth.py`, `accounts/microsoft_auth.py`, `accounts/social_auth.py`, `accounts/tests/test_google_auth.py`, `accounts/tests/test_microsoft_auth.py`, `deploy/systemd/app-backend-expire-trials.timer`
- Modify: `config/settings.py`, `config/urls.py`, `accounts/auth_views.py`, `accounts/urls.py`, `accounts/auth_serializers.py`, `accounts/models/user_profile.py`, `accounts/serializers/user_profile.py`, `rbac/catalog.py`, `rbac/models/audit_log.py`, `.env.example`, `requirements.txt`, `.github/workflows/ci.yml`, `docker/release.sh`, `deploy/systemd/README.md`, `README.md`

**Interfaces:**
- Produces: a template copy that passes its own checks with no social sign-in and no billing.

- [ ] **Step 1: Copy template, create venv and database**

Run: `git clone --depth 1 https://github.com/abdalrhmanreda/backend-setup /tmp/tpl && rsync -a --exclude .git --exclude docs/superpowers /tmp/tpl/ ./ && python3 -m venv venv && venv/bin/pip install -r requirements-dev.txt && createdb katalog && cp .env.example .env`
Set in `.env`: `APP_NAME=Katalog`, `DJANGO_DEBUG=true`, `POSTGRES_DB=katalog`.

- [ ] **Step 2: Remove social sign-in and billing**

Delete the files listed above. Remove: `GoogleLoginView`, `MicrosoftLoginView`, `SocialLoginView` and their imports and routes; `SocialLoginSerializer`; `google_sub`, `microsoft_sub`, `google_connected` (model + serializer); `GOOGLE_*`/`MICROSOFT_*` settings and env lines; `'billing'` from `INSTALLED_APPS` and its include; `PAYMENT_BANK_DETAILS` block; `plans`/`payments` resources, permissions and role slugs in `rbac/catalog.py`; `PLAN_*`, `PAYMENT_*`, `SUBSCRIPTION_*` audit actions; `google-auth` from `requirements.txt`; `seed_plans`/`expire_trials` from `docker/release.sh`, CI and systemd README. Keep `PRIVATE_MEDIA_ROOT` (media uses it in Task 8). Add `drf-spectacular==0.28.0` to `requirements.txt`.

- [ ] **Step 3: Regenerate the touched migrations**

Run: `venv/bin/python manage.py makemigrations accounts rbac` (they will be thrown away in Task 3; this keeps checks green now).

- [ ] **Step 4: Run checks**

Run: `venv/bin/ruff check . && venv/bin/python manage.py check && venv/bin/python manage.py makemigrations --check --dry-run && venv/bin/python manage.py test`
Expected: ruff clean, `System check identified no issues`, `No changes detected`, tests `OK`.

---

### Task 2: RAML envelope, `/api/v1/` mounting, optional trailing slash

**Files:**
- Modify: `core/renderers.py`, `core/exceptions.py`, `config/urls.py`, `config/settings.py`, every app `urls.py`
- Create: `core/routing.py`, `core/views_errors.py`, `core/tests/test_envelope.py`

**Interfaces:**
- Produces: `core.routing.route(pattern: str, view, name: str | None = None) -> list[URLPattern]` returning the pattern with and without trailing `/` (name only on the slash-less one). Every app `urls.py` from here on builds `urlpatterns` with `route(...)` and the pattern written without a trailing slash.
- Produces: `core.exceptions.api_error(message: str, code: str, status: int, details: dict | None = None) -> Response` for views that refuse with a custom code.
- Produces: error body `{"message", "code", "details"?}`. `ValidationError` → message `"Invalid request data."`, code `"validation_error"`, `details` = serializer errors. Other `APIException` → message = `str(detail)` (or `detail["detail"]` when dict), code = first `get_codes()` value or `default_code`. Dict detail with a `code` key keeps that code (template `EmailNotVerified`).

- [ ] **Step 1: Write failing tests in `core/tests/test_envelope.py`**

```python
def test_success_envelope(self):
    r = self.client.get("/api/v1/health")
    assert r.status_code == 200
    assert set(r.json()) == {"status_code", "data", "error"}
    assert r.json()["error"] is None and r.json()["status_code"] == 200

def test_validation_error_envelope(self):
    r = self.client.post("/api/v1/auth/login", {}, content_type="application/json")
    body = r.json()
    assert r.status_code == 400 and body["data"] is None
    assert body["error"]["code"] == "validation_error"
    assert "email" in body["error"]["details"]

def test_unauthenticated_envelope(self):
    r = self.client.get("/api/v1/me")
    assert r.status_code == 401
    assert r.json()["error"]["code"] in ("not_authenticated", "unauthorized")

def test_unknown_url_is_json_404(self):
    r = self.client.get("/api/v1/nope")
    assert r.status_code == 404
    assert r.json() == {"status_code": 404, "data": None, "error": {"message": "Not found.", "code": "not_found"}}

def test_trailing_slash_optional_for_post(self):
    a = self.client.post("/api/v1/auth/login", {}, content_type="application/json")
    b = self.client.post("/api/v1/auth/login/", {}, content_type="application/json")
    assert a.status_code == b.status_code == 400
```

(`/api/v1/auth/login` and `/api/v1/me` exist after Task 4/5; until then point the template's current login and me routes at these paths in `accounts/urls.py`.)

- [ ] **Step 2: Run** `venv/bin/python manage.py test core.tests.test_envelope` — Expected: FAIL (`errors` key, 404 HTML, missing prefix).

- [ ] **Step 3: Implement**

Renderer emits `error`; already-wrapped detection checks the `error` key. Exception handler builds the error object described above. `handler404`/`handler500` in `config/urls.py` point at `core.views_errors` JSON views (500 message `"Server error."`, code `"server_error"`). Mount: `admin/` stays; every app under `api/v1/` (`core`, `accounts`, `rbac`, `legal`, `notifications` at `api/v1/notifications/`, `support` at `api/v1/support/`). Drop the `/api/core/` aliases. Set `APPEND_SLASH = False`. Convert every existing `urls.py` to `route(...)`. Add drf-spectacular: `INSTALLED_APPS += ["drf_spectacular"]`, `DEFAULT_SCHEMA_CLASS`, routes `api/v1/schema` and `api/v1/docs` (Swagger UI). Update template tests that assert `errors` or old paths.

- [ ] **Step 4: Run** `venv/bin/python manage.py test` — Expected: OK.

---

### Task 3: Custom `User` model (UUID, email login), remove `UserProfile`

**Files:**
- Create: `accounts/models/user.py`, `accounts/managers.py`, `accounts/tests/test_user_model.py`
- Delete: `accounts/models/user_profile.py`, `accounts/signals.py`, `accounts/management/commands/backfill_profiles.py`, every `*/migrations/0*.py`
- Modify: `accounts/models/__init__.py`, `accounts/apps.py`, `accounts/admin.py`, `accounts/admin_views.py`, `accounts/permissions.py`, `accounts/serializers/*`, `accounts/auth_serializers.py`, `accounts/auth_views.py`, `rbac/services.py`, `rbac/views.py`, `rbac/admin.py`, `rbac/audit.py`, `core/log_serializers.py`, `core/admin_views.py`, `core/models/idempotency.py`, `notifications/models.py`, `support/models.py`, `support/*views.py`, all `*/urls.py` with `<int:pk>` for users, template tests, `config/settings.py`

**Interfaces:**
- Produces: `accounts.models.User(AbstractUser)`, `AUTH_USER_MODEL = "accounts.User"`, `USERNAME_FIELD = "email"`, `REQUIRED_FIELDS = ["full_name"]`, `username = None`, `first_name = None`, `last_name = None`.
  Fields (ERD): `id` UUID pk default `uuid4`; `full_name` Char(150); `email` unique; `phone` Char(32) blank; `avatar_url` URL(500) blank; `pin_hash` Char(128) blank; `email_verified_at` DateTime null; `locale` Char(10) default `"en"`; `timezone` Char(64) default `"UTC"`; `last_active_at` DateTime default now; `status` choices `UserStatus` = `active|suspended|deceased_confirmed` default `active`; `current_plan` FK `"plans.Plan"` null, SET_NULL (add after Task 5 creates `plans`; leave a `# Task 5` note now); `deleted_at` DateTime null; `created_at` auto_now_add; `updated_at` auto_now. Indexes: `email`, `last_active_at`.
  Properties: `has_pin -> bool`, `is_email_verified -> bool`, `is_deleted -> bool`. Methods: `set_pin(raw: str) -> None`, `check_pin(raw: str) -> bool` (Django `make_password`/`check_password`).
- Produces: `accounts.managers.UserManager.create_user(email, password=None, **extra)` and `create_superuser(email, password, **extra)`; email normalized lower-case.
- Produces: `accounts.serializers.UserSerializer` with exactly the RAML `User` fields: `id, full_name, email, phone, avatar_url, has_pin, email_verified_at, locale, timezone, last_active_at, status, created_at`. Used by every later task that returns a user.

- [ ] **Step 1: Write failing tests in `accounts/tests/test_user_model.py`**

```python
def test_create_user_by_email(self):
    u = User.objects.create_user(email="A@X.com", password="pw-123456", full_name="A")
    assert isinstance(u.pk, uuid.UUID) and u.email == "a@x.com" and u.check_password("pw-123456")
    assert u.status == "active" and u.locale == "en" and u.timezone == "UTC" and not u.has_pin

def test_pin_is_hashed(self):
    u = User.objects.create_user(email="b@x.com", password="pw-123456", full_name="B")
    u.set_pin("1234"); u.save()
    assert u.pin_hash != "1234" and u.check_pin("1234") and not u.check_pin("0000") and u.has_pin

def test_user_serializer_fields(self):
    data = UserSerializer(User.objects.create_user(email="c@x.com", password="pw-123456", full_name="C")).data
    assert set(data) == {"id","full_name","email","phone","avatar_url","has_pin","email_verified_at","locale","timezone","last_active_at","status","created_at"}
    assert isinstance(data["id"], str)
```

- [ ] **Step 2: Run** `venv/bin/python manage.py test accounts.tests.test_user_model` — Expected: FAIL (import error).

- [ ] **Step 3: Implement the model, swap references, regenerate migrations**

Replace `profile.display_name` → `full_name`, `profile.is_verified` → `is_email_verified`, `profile.deleted_at` → `deleted_at`, `user.username`/`user__username` → `get_username()`/`email`. URL converters for user ids become `<uuid:pk>`; `rbac/views.py` actor filter accepts a UUID or an email. `rbac.AuditLog.actor_username` keeps its name, stores email. Rebuild `rbac/migrations/0002_seed_catalog.py` from the template version with `apps.get_model(settings.AUTH_USER_MODEL)` and a dependency on `migrations.swappable_dependency(settings.AUTH_USER_MODEL)`. Then: `dropdb katalog && createdb katalog && venv/bin/python manage.py makemigrations accounts core rbac notifications legal support` (then hand-add the 0002 data migration). Template admin user endpoints keep working on the new fields.

- [ ] **Step 4: Run** `venv/bin/python manage.py migrate && venv/bin/python manage.py test` — Expected: OK (template tests updated for email login and UUIDs).

---

### Task 4: Auth endpoints per RAML

**Files:**
- Modify: `accounts/auth_views.py`, `accounts/auth_serializers.py`, `accounts/urls.py`, `accounts/models/verification_code.py`, `accounts/throttles.py`, `config/settings.py`, `.env.example`
- Delete: `accounts/auth_cookies.py`, `accounts/tests/test_auth_cookies.py`
- Create: `accounts/otp.py`, `accounts/tests/test_raml_auth.py`

**Interfaces:**
- Consumes: `UserSerializer`, `User`, `core.exceptions.api_error`, `core.routing.route`.
- Produces: `VerificationPurpose` values `verify_email`, `reset_password`, `recipient_access`.
- Produces in `accounts/otp.py`: `send_otp(email: str, purpose: str) -> None` (generates and emails; caller decides whether to call), `check_otp(email: str, code: str, purpose: str, *, spend: bool) -> VerificationCode` raising `OtpRefused(reason: str, message: str)` where `reason` ∈ `no_code|expired|attempts_exhausted|invalid` (moved from template `_spend_code`, wrong guesses always counted). Views turn `OtpRefused` into `api_error(message, code=reason, status=400)`.
- Produces: `accounts.services.blacklist_all_refresh_tokens(user) -> None`.
- Produces: signal hook `accounts.signals.user_registered` (Django `Signal`) sent with `user=` after register; Tasks 5 and 6 connect to it.
- Routes (all under `/api/v1/`): `auth/register`, `auth/login`, `auth/refresh`, `auth/logout`, `auth/otp/send`, `auth/otp/verify`, `auth/password/forgot`, `auth/password/reset`. Template `auth/send-verification-code`, `verify-code`, `password/reset/verify`, `password/reset/confirm`, `password/change`, `me/profile`, `auth/me` are removed.

- [ ] **Step 1: Write failing tests in `accounts/tests/test_raml_auth.py`**

```python
def test_register_returns_user_and_tokens(self):
    r = post("/api/v1/auth/register", {"full_name": "Sara", "email": "s@x.com", "password": "Strong-pass-1", "timezone": "Africa/Cairo"})
    assert r.status_code == 201
    d = r.json()["data"]; assert set(d) == {"user", "access", "refresh"}
    assert d["user"]["email"] == "s@x.com" and d["user"]["timezone"] == "Africa/Cairo"
    assert len(mail.outbox) == 1  # verify_email code

def test_register_duplicate_email_400(self): ...  # second register same email -> 400, error.code == "validation_error", "email" in details

def test_login_bad_password_400(self):
    r = post("/api/v1/auth/login", {"email": "s@x.com", "password": "wrong"})
    assert r.status_code == 400 and r.json()["error"]["code"] == "invalid_credentials"

def test_login_updates_last_active_at(self): ...

def test_refresh_returns_access_and_token_is_reusable(self):
    refresh = login()["refresh"]
    for _ in range(2):
        r = post("/api/v1/auth/refresh", {"refresh": refresh})
        assert r.status_code == 200 and set(r.json()["data"]) == {"access"}

def test_logout_blacklists_refresh(self):
    # authed POST /auth/logout {"refresh"} -> 200 data.detail == "Successfully logged out."
    # then /auth/refresh with it -> 401

def test_otp_send_same_answer_for_unknown_email(self):
    a = post("/api/v1/auth/otp/send", {"target": "nobody@x.com", "purpose": "verify_email"})
    b = post("/api/v1/auth/otp/send", {"target": "s@x.com", "purpose": "verify_email"})
    assert a.json() == b.json() and a.json()["data"] == {"message": "OTP sent successfully.", "expires_in_seconds": 300}

def test_otp_verify_email_sets_verified(self): ...  # data == {"verified": True}; user.email_verified_at not None

def test_otp_verify_wrong_code(self): ...  # 400, error.code == "invalid"

def test_otp_verify_reset_password_returns_reset_token(self): ...  # data["verified"] is True and data["reset_token"]

def test_password_reset_with_code(self):
    # forgot -> code from mail.outbox -> POST /auth/password/reset {"email","code","new_password"}
    # -> 200 data.detail == "Password reset successfully."; old refresh tokens rejected; login with new password works

def test_password_reset_with_reset_token(self): ...  # code=<reset_token from otp/verify> also works once

def test_forgot_unknown_email_same_answer(self): ...  # data == {"message": "Password reset OTP sent to email."}
```

- [ ] **Step 2: Run** `venv/bin/python manage.py test accounts.tests.test_raml_auth` — Expected: FAIL (404s).

- [ ] **Step 3: Implement**

Settings: `ROTATE_REFRESH_TOKENS=False`, `BLACKLIST_AFTER_ROTATION=False`, remove `AUTH_REFRESH_IN_BODY` and cookie code (refresh always in body). Login serializer takes `email`, `password`; wrong credentials or inactive/deleted user → 400 `invalid_credentials`, message `"Invalid email or password."`. Register accepts `full_name, email, password, phone?, locale?, timezone?`, validates password with Django validators and timezone with `zoneinfo`. Logout: `IsAuthenticated`, 200 `{"detail": "Successfully logged out."}`. Keep template throttle classes on each endpoint (send: burst + per-email; verify: per IP + per email; login: per IP + per identifier). `recipient_access` send rule from spec is implemented in Task 9 by connecting `accounts.otp.recipient_access_allowed` — here define `recipient_access_allowed: Callable[[str], bool]` defaulting to `lambda email: False`, and let Task 9 replace it via `messaging.apps.ready()`.

- [ ] **Step 4: Run** `venv/bin/python manage.py test` — Expected: OK.

---

### Task 5: `plans` app, `/plans`, `/me`, `/me/pin`, `/me/subscription`, `/devices`

**Files:**
- Create: `plans/` (models, admin, serializers, views, urls, `management/commands/seed_plans.py`, tests), `accounts/models/device.py`, `accounts/me_views.py`, `accounts/tests/test_me.py`
- Modify: `accounts/models/user.py` (add `current_plan`), `accounts/urls.py`, `config/settings.py`, `config/urls.py`, `docker/release.sh`, `accounts/auth_views.py` (assign free plan on register)

**Interfaces:**
- Produces: `plans.models.Plan` per ERD (`id` UUID, `code` unique choices `free|monthly|one_time|business`, `name`, `price` Decimal(10,2), `currency` Char(3) default `"USD"`, `billing_period` Char(20) blank, `max_messages`, `storage_years`, `max_media_mb` PositiveInteger, `features` JSON default dict, `is_active` bool, `created_at`).
- Produces: `plans.services.plan_for(user) -> Plan` (user's plan, else the `free` plan), `plans.services.messages_used(user) -> int` (implemented in Task 7, returns 0 until then), `plans.services.storage_used_bytes(user) -> int` (Task 8, returns 0 until then).
- Produces: `accounts.signals.user_deleted` (Django `Signal`, sent with `user=` inside the `DELETE /me` transaction). Tasks 6, 7 and 8 connect receivers that soft-delete contacts, soft-delete messages and delete media files.
- Produces: `accounts.models.Device` (`id` UUID, `user` FK CASCADE, `fcm_token` unique Char(512), `platform` `ios|android|web`, `last_seen_at`, `created_at`).
- Routes: `plans` (GET, AllowAny, active plans ordered by price), `me` (GET/PATCH/DELETE), `me/pin` (POST), `me/pin/verify` (POST), `me/subscription` (GET), `devices` (POST). `me/home` comes in Task 9.

- [ ] **Step 1: Write failing tests (`plans/tests/test_plans.py`, `accounts/tests/test_me.py`)**

```python
def test_plans_public_list(self):
    call_command("seed_plans")
    d = self.client.get("/api/v1/plans").json()["data"]
    assert [p["code"] for p in d] == ["free", "monthly", "one_time", "business"]
    assert set(d[0]) == {"id","code","name","price","currency","billing_period","max_messages","storage_years","max_media_mb","features","is_active"}

def test_register_assigns_free_plan(self): ...

def test_me_get_and_patch(self):
    # PATCH {"full_name": "New", "locale": "ar", "timezone": "Asia/Riyadh"} -> 200 user with new values
    # PATCH {"email": "x@y.com"} -> email unchanged (not writable)
    # PATCH {"timezone": "Mars/Base"} -> 400 validation_error

def test_me_delete_soft_deletes(self):
    # DELETE /me -> 200 data.detail == "Account deleted successfully."
    # user.deleted_at set, is_active False, email startswith "deleted+"; login with old creds -> 400; old refresh -> 401

def test_pin_set_change_verify(self):
    # POST /me/pin {"pin": "1234"} -> 200 detail "PIN saved successfully."; GET /me has_pin True
    # POST /me/pin {"pin": "5678"} without old_pin -> 400 code "old_pin_required"
    # POST /me/pin {"pin": "5678", "old_pin": "0000"} -> 400 code "invalid_pin"
    # POST /me/pin {"pin": "12"} -> 400 validation_error; {"pin": "12ab"} -> 400 (digits only, 4-6)
    # POST /me/pin/verify {"pin": "1234"} -> data == {"valid": True}; wrong -> {"valid": False}

def test_subscription(self):
    # GET /me/subscription -> data keys {"plan","status","messages_remaining","storage_remaining_mb"}; status == "active"
    # free plan: messages_remaining == plan.max_messages

def test_device_register_upsert(self):
    # POST /devices {"fcm_token": "t1", "platform": "android"} -> 201 detail "Device registered successfully."
    # same token from another user -> 201, row now belongs to the second user (one row)
    # platform "symbian" -> 400
```

- [ ] **Step 2: Run** `venv/bin/python manage.py test plans accounts.tests.test_me` — Expected: FAIL.

- [ ] **Step 3: Implement**

Seed values (placeholders, editable in admin): free 0.00 / 3 messages / 1 year / 100 MB; monthly 4.99 `month` / 50 / 5 / 2048; one_time 49.00 `once` / 100 / 25 / 5120; business 0.00 `custom` / 1000 / 50 / 51200. `seed_plans` creates only missing codes; add it to `docker/release.sh`. `PATCH /me` writable fields: `full_name, avatar_url, locale, timezone`. PIN verify uses a scoped throttle `pin_verify` = `10/hour` per user. Delete account audit-recorded (`USER_DELETED`).

- [ ] **Step 4: Run** `venv/bin/python manage.py test` — Expected: OK.

---

### Task 6: `contacts` app

**Files:**
- Create: `contacts/` (models, admin, serializers, views, urls, services, tests)
- Modify: `config/settings.py`, `config/urls.py`

**Interfaces:**
- Consumes: `accounts.signals.user_registered` (receiver calls `link_contacts_for`), `accounts.signals.user_deleted` (receiver soft-deletes the user's contacts).
- Produces: `contacts.models.Contact` per ERD (`id` UUID, `owner` FK CASCADE `related_name="contacts"`, `name` Char(150), `relation` Char(60) blank, `phone` Char(32) blank, `email` Email blank, `is_self` bool, `linked_user` FK null SET_NULL `related_name="linked_contacts"`, `last_used_at` null, `deleted_at` null, timestamps). Indexes `(owner, name)`, `(owner, last_used_at)`. Unique constraint: one `is_self=True` per owner where `deleted_at` is null.
- Produces: `Contact.objects.live()` (not deleted); `contacts.services.link_contacts_for(user) -> int` (sets `linked_user` on all live contacts with that email); `contacts.services.resolve_linked_user(email: str) -> User | None`.
- Produces: `ContactSerializer` with RAML fields `id, owner_id, name, relation, email, phone, is_self, linked_user_id, last_used_at, created_at` (ids as strings).
- Routes: `contacts` (GET `?search=`, POST), `contacts/recent` (GET), `contacts/import` (POST), `contacts/<uuid:id>` (PATCH, DELETE).

- [ ] **Step 1: Write failing tests (`contacts/tests/test_contacts.py`)**

```python
def test_create_and_list(self):
    # POST {"name":"Fatima","relation":"Mother","email":"fatima@example.com"} -> 201, data["is_self"] False
    # GET /contacts -> list of 1; GET /contacts?search=fat -> 1; ?search=zzz -> 0

def test_contacts_are_private(self):
    # user B GET /contacts -> []; B PATCH/DELETE A's contact -> 404

def test_linked_user_set_by_email(self):
    # contact email == existing user's email -> linked_user_id == that user's id
    # contact created first, then that email registers -> linked after register

def test_single_self_contact(self):
    # POST is_self True -> 201, linked_user_id == owner id; second is_self -> 400 code "self_contact_exists"

def test_delete_is_soft(self):
    # DELETE -> 200 detail "Contact deleted successfully."; GET list excludes it; row still exists with deleted_at

def test_recent(self):
    # 12 contacts with last_used_at set + 1 without -> /contacts/recent returns 10, newest first, none without last_used_at

def test_import_skips_duplicates(self):
    # existing email a@x.com; import [{"name":"A","email":"a@x.com"},{"name":"B","phone":"+201"},{"name":"C"}]
    # -> 201 data["imported_count"] == 2, len(data["contacts"]) == 2
    # 501 items -> 400 validation_error
```

- [ ] **Step 2: Run** `venv/bin/python manage.py test contacts` — Expected: FAIL.
- [ ] **Step 3: Implement** — `IsAuthenticated` on all; querysets filter `owner=request.user`; search over `name`, `relation`, `email` (icontains).
- [ ] **Step 4: Run** `venv/bin/python manage.py test` — Expected: OK.

---

### Task 7: `messaging` app — encryption, messages CRUD, actions, message types

**Files:**
- Create: `messaging/` (`models.py`, `crypto.py`, `serializers.py`, `views.py`, `urls.py`, `message_types.py`, `admin.py`, `tests/test_crypto.py`, `tests/test_messages.py`)
- Modify: `config/settings.py` (`MESSAGE_ENCRYPTION_KEY`), `.env.example`, `plans/services.py` (`messages_used`), `config/urls.py`

**Interfaces:**
- Consumes: `accounts.signals.user_deleted` (receiver soft-deletes the user's messages).
- Produces in `messaging/crypto.py`: `encrypt(plaintext: str) -> tuple[str, str, str]` returning base64 `(ciphertext, iv, tag)` with a fresh 12-byte IV; `decrypt(ciphertext: str, iv: str, tag: str) -> str`. Uses `cryptography.hazmat.primitives.ciphers.aead.AESGCM`.
- Produces models (ERD):
  - `Message`: `id` UUID, `sender` FK CASCADE `related_name="messages"`, `type` `dua|advice|love|apology|reassurance`, `title` Char(200), `body_encrypted` Text, `body_iv` Char(32), `body_tag` Char(32), `status` `draft|scheduled|queued|sent|opened|paused|failed` default draft, `trigger_type` `instant|date|occasion|inactivity`, `deliver_at` null, `occasion_type` `birth|wedding|engagement|graduation` blank, `occasion_note` Char(255) blank, `channel` `in_app|email`, `is_saved_moment` bool, `paused_at`, `confirmed_at`, `sent_at`, `deleted_at` null, timestamps. Indexes `(sender, status)`, `(status, deliver_at)`. Property `body` (decrypt) and method `set_body(text: str)`.
  - `MessageRecipient`: `id` UUID, `message` FK CASCADE, `contact` FK CASCADE, `created_at`; unique `(message, contact)`.
  - `MessageMedia` and `Delivery` are defined in Tasks 8 and 9.
- Produces: `MessageSerializer` output fields `id, sender_id, type, title, body, status, trigger_type, deliver_at, occasion_type, occasion_note, channel, is_saved_moment, paused_at, confirmed_at, sent_at, created_at, media, recipient_ids` (`media` = `[]` until Task 8).
- Produces: `messaging.services.confirm_message(message) -> Message`, hook point `release_message(message, now)` (Task 9 fills it; until then `instant` confirm sets status `sent`, `sent_at`).
- Routes: `messages` (GET `?status=`, `?is_saved=true|false`; POST), `messages/<uuid:id>` (GET, PATCH, DELETE), `messages/<uuid:id>/confirm|pause|resume` (POST), `messages/<uuid:id>/saved` (PATCH `{"is_saved": bool}`), `message-types` (GET, AllowAny).

- [ ] **Step 1: Write failing tests**

```python
# test_crypto.py
def test_round_trip_and_random_iv(self):
    c1 = encrypt("I am always proud of you"); c2 = encrypt("I am always proud of you")
    assert decrypt(*c1) == "I am always proud of you" and c1[1] != c2[1]
def test_tampered_tag_fails(self): ...  # raises cryptography.exceptions.InvalidTag
def test_missing_key_without_debug_is_improperly_configured(self): ...

# test_messages.py
def test_create_draft(self):
    # POST {"type":"advice","title":"For your graduation day","body":"I am always proud of you...",
    #       "recipient_ids":[contact_id],"trigger_type":"date","deliver_at":"2027-06-01T10:00:00Z","channel":"email"}
    # -> 201, data["status"] == "draft", data["body"] == the text, data["recipient_ids"] == [contact_id]
    # DB row: body_encrypted != text and "proud" not in body_encrypted

def test_validation(self):
    # trigger date without deliver_at -> 400; deliver_at in the past -> 400
    # trigger occasion without occasion_type -> 400
    # recipient_ids containing another user's contact or a deleted contact -> 400
    # empty recipient_ids -> 400

def test_list_filters(self): ...  # ?status=draft, ?is_saved=true; deleted messages excluded; other user's excluded

def test_other_users_message_404(self): ...  # GET/PATCH/DELETE/confirm -> 404

def test_confirm_date_message(self):
    # POST confirm -> 200 status "scheduled", confirmed_at set; recipient contact last_used_at set
def test_confirm_twice_400(self): ...  # error.code == "invalid_state"
def test_confirm_respects_plan_limit(self): ...  # free plan max 3: 4th confirm -> 400 "plan_limit_reached"

def test_pause_resume(self):
    # scheduled -> pause -> "paused", paused_at set -> resume -> "scheduled", paused_at None
    # pause a draft -> 400 invalid_state

def test_edit_only_in_editable_states(self):
    # PATCH title on draft/scheduled/paused -> 200; after status "sent" -> 400 invalid_state
    # PATCH recipient_ids replaces recipients

def test_saved_toggle(self): ...  # PATCH saved {"is_saved": true} -> data["is_saved_moment"] True
def test_delete_soft(self): ...  # detail "Message deleted successfully."; excluded from list

def test_message_types(self):
    d = self.client.get("/api/v1/message-types").json()["data"]
    assert [t["key"] for t in d] == ["dua","advice","love","apology","reassurance"]
    assert set(d[0]) == {"key","name","icon","placeholder"}
    # Accept-Language: ar -> Arabic names
```

- [ ] **Step 2: Run** `venv/bin/python manage.py test messaging` — Expected: FAIL.
- [ ] **Step 3: Implement** — `plans.services.messages_used` counts sender's messages with `deleted_at` null and status ≠ draft.
- [ ] **Step 4: Run** `venv/bin/python manage.py test` — Expected: OK.

---

### Task 8: Media upload, signed download, attach to messages

**Files:**
- Create: `messaging/media.py` (storage + signing), `messaging/media_views.py`, `messaging/tests/test_media.py`
- Modify: `messaging/models.py`, `messaging/serializers.py`, `messaging/urls.py`, `plans/services.py` (`storage_used_bytes`)

**Interfaces:**
- Produces: `MessageMedia` (ERD + `owner` FK CASCADE, `uploaded_at` null): `id` UUID, `message` FK null CASCADE `related_name="media"`, `kind` `voice|image|video`, `storage_key` Char(255), `mime_type`, `size_bytes` PositiveBigInteger, `duration_sec` null, `sort_order`, `created_at`.
- Produces in `messaging/media.py`: `storage_path(media) -> Path` (`PRIVATE_MEDIA_ROOT / "media" / str(owner_id) / str(id)`), `signed_url(request, media) -> str`, `check_signature(media_id: str, sig: str) -> bool` (`django.core.signing.TimestampSigner`, max age 3600 s), `delete_file(media) -> None`.
- Produces: `MessageMediaSerializer` RAML fields `id, kind, storage_key, url, mime_type, size_bytes, duration_sec, sort_order` (`url` null until uploaded).
- Routes: `media/upload-url` (POST), `media/<uuid:id>/upload` (PUT raw body), `media/<uuid:id>/confirm` (POST), `media/<uuid:id>` (DELETE), `media/<uuid:id>/file` (GET, AllowAny, needs valid `sig`; streams `FileResponse`).
- Message create/PATCH accept `media_ids`: media must belong to the caller, be uploaded, and be unattached or attached to this message; list order sets `sort_order`.

- [ ] **Step 1: Write failing tests (`messaging/tests/test_media.py`, use `override_settings(PRIVATE_MEDIA_ROOT=tmpdir)`)**

```python
def test_full_upload_flow(self):
    # POST upload-url {"kind":"voice","file_name":"a.m4a","mime_type":"audio/mp4","size_bytes":5}
    # -> 200 data keys {"media_id","upload_url","storage_key"}; upload_url ends with f"/api/v1/media/{id}/upload"
    # PUT upload_url body b"hello" content_type "audio/mp4" -> 200
    # POST confirm {"duration_sec": 3} -> 200 data["duration_sec"] == 3 and data["url"] has "sig="
    # GET data["url"] without auth -> 200 and content == b"hello"

def test_bad_signature_404(self): ...
def test_kind_mime_mismatch_400(self): ...   # kind image, mime audio/mp4 -> 400
def test_size_mismatch_400(self): ...        # declared 5, PUT 6 bytes -> 400 code "size_mismatch"
def test_storage_limit(self): ...            # size_bytes > plan max_media_mb*1024*1024 - used -> 400 "storage_limit_reached"
def test_other_user_cannot_upload_or_attach(self): ...  # PUT/confirm/DELETE -> 404; media_ids with foreign media -> 400
def test_confirm_before_upload_400(self): ...  # code "not_uploaded"
def test_delete_removes_file(self): ...      # detail "Media deleted successfully."; file gone
def test_message_returns_media_in_order(self): ...
```

- [ ] **Step 2: Run** `venv/bin/python manage.py test messaging.tests.test_media` — Expected: FAIL.
- [ ] **Step 3: Implement** — `storage_key` = `media/<owner>/<id>`; Connect a `user_deleted` receiver that calls `delete_file` for each of the user's media.
- [ ] **Step 4: Run** `venv/bin/python manage.py test` — Expected: OK.

---

### Task 9: Release, deliveries, inbox, public recipient link, home

**Files:**
- Create: `messaging/services.py` (fill), `messaging/delivery_views.py`, `messaging/public_views.py`, `messaging/home_views.py`, `messaging/emails.py`, `messaging/tests/test_release.py`, `messaging/tests/test_inbox.py`, `messaging/tests/test_public_link.py`, `messaging/tests/test_home.py`
- Modify: `messaging/models.py`, `messaging/apps.py`, `messaging/urls.py`, `notifications/models.py` (add types `message_received`, `message_opened`, `checkin`, `trustee`)

**Interfaces:**
- Produces: `Delivery` per ERD (`id` UUID, `message` FK CASCADE `related_name="deliveries"`, `contact` FK CASCADE, `channel`, `status` `queued|sent|delivered|failed|opened`, `access_token_hash` Char(64) unique, `attempts`, `last_error` Text blank, `sent_at`, `opened_at`, `open_later_at`, timestamps). Indexes `access_token_hash`, `(contact, status)`.
- Produces in `messaging/services.py`:
  - `release_message(message, now) -> list[tuple[Delivery, str]]` — inside a transaction: one Delivery per live recipient, raw token returned with each; status `sent`, `sent_at=now`; email channel sends `{FRONTEND_BASE_URL}/r/{token}` (inline when `EMAIL_SEND_IN_BACKGROUND` is off) and marks the delivery `sent` or `failed`; in_app marks `delivered`; notification to `contact.linked_user` (type `message_received`, data `{"delivery_id": ...}`); `rbac.audit.record`-style entry is not needed here.
  - `release_due_messages(now=None) -> int` — `status="scheduled"`, `trigger_type in (date, occasion)`, `deliver_at <= now`, `deleted_at` null; `select_for_update(skip_locked=True)`.
  - `release_inactivity_messages(user, now) -> int` (Task 10 calls it).
  - `delivery_for_token(token: str) -> Delivery` (404 when unknown, message deleted, or not released).
  - `mark_opened(delivery, now) -> None` (delivery `opened`, `opened_at`; message status `opened` on first open; notification `message_opened` to sender).
  - `confirm_message` now releases `instant` messages via `release_message`.
- Produces: `accounts.otp.recipient_access_allowed` replaced in `messaging.apps.MessagingConfig.ready()` with: a live contact with that email has a delivery in `sent|delivered|opened`.
- Produces: view session token = `signing.dumps({"d": delivery_id}, salt="recipient-view")`, max age 1800 s.
- Routes: `inbox` (GET), `inbox/<uuid:delivery_id>` (GET; marks opened), `r/<str:token>` (GET), `r/<str:token>/verify` (POST), `r/<str:token>/open` (POST), `r/<str:token>/later` (POST), `me/home` (GET).

- [ ] **Step 1: Write failing tests**

```python
# test_release.py
def test_instant_confirm_releases_email(self):
    # contact with email, channel email, trigger instant -> confirm -> status "sent"
    # one Delivery status "sent", mail.outbox[-1] body contains f"{FRONTEND_BASE_URL}/r/"
    # Delivery.access_token_hash == sha256(token from email).hexdigest(); token itself not stored anywhere
def test_due_date_message_released_on_request(self):
    # scheduled date message, deliver_at moved to past via queryset update -> GET /api/v1/inbox as linked user -> item present
def test_future_message_not_released(self): ...
def test_paused_message_not_released(self): ...
def test_deleted_contact_gets_no_delivery(self): ...
def test_email_failure_marks_failed(self): ...  # patch send_mail to raise -> delivery "failed", last_error set, message still "sent"
def test_release_is_idempotent(self): ...       # release_due_messages twice -> deliveries count unchanged

# test_inbox.py
def test_inbox_lists_linked_deliveries(self):
    # item keys == {"delivery_id","message_id","sender_name","sender_avatar","title","type","channel","status","created_at","delivered_at"}
def test_inbox_detail_decrypts_and_marks_opened(self):
    # keys {"delivery_id","sender_name","type","title","body","media","sent_at","opened_at"}; body plaintext; delivery "opened"; message "opened"
def test_inbox_detail_other_user_404(self): ...
def test_self_message_reaches_own_inbox(self): ...  # is_self contact, in_app

# test_public_link.py
def test_lookup(self):
    # GET /r/<token> -> {"sender_name","requires_verification": True,"title"}; unknown token -> 404
def test_open_requires_verification(self):
    # POST /r/<token>/open without session -> 400 code "verification_required"
    # otp/send recipient_access to contact email -> code mailed; POST /r/<token>/verify {"email","code"} -> {"verified": True,"view_session_token"}
    # POST /r/<token>/open {"view_session_token"} -> body, media, opened_at
def test_verify_wrong_email_400(self): ...     # email != contact email -> 400 "email_mismatch"
def test_session_token_bound_to_delivery(self): ...  # token for delivery A refused on delivery B
def test_later(self): ...                      # detail "Marked to open later.", open_later_at set
def test_recipient_otp_not_sent_without_delivery(self): ...  # same 200, no mail

# test_home.py
def test_home(self):
    # keys {"user","unread_notifications_count","has_waiting_message","waiting_messages_count","recent_contacts","scheduled_messages_count"}
    # waiting = caller's inbox deliveries not opened; scheduled = caller's messages in scheduled|paused
```

- [ ] **Step 2: Run** `venv/bin/python manage.py test messaging` — Expected: FAIL.
- [ ] **Step 3: Implement** — call `release_due_messages()` at the start of inbox, inbox detail, home, message list/detail and `r/*` views. `sender_name` = sender `full_name`; `delivered_at` = delivery `sent_at`.
- [ ] **Step 4: Run** `venv/bin/python manage.py test` — Expected: OK.

---

### Task 10: `inactivity` app, trustee link, `process_due`

**Files:**
- Create: `inactivity/` (models, admin, serializers, services, views, urls, emails, `management/commands/process_due.py`, tests), `deploy/systemd/app-backend-process-due.timer`
- Modify: `config/settings.py`, `config/urls.py`, `deploy/systemd/README.md`, `accounts/auth_views.py` (login does not reset check-ins; only `last_active_at`)

**Interfaces:**
- Consumes: `messaging.services.release_inactivity_messages(user, now)`, `messaging.services.release_due_messages(now)`, `Contact`.
- Produces models (ERD): `InactivitySettings` (`user` OneToOne primary key, `enabled`, `checkin_interval_days`=30, `missed_checkins_limit`=3, `grace_period_days`=7, `verify_by_email`=True, `trustee_contact` FK null SET_NULL, `state` `active|warning|pending_trustee|released|cancelled`, `next_checkin_at` null, `limit_reached_at` null (builder addition for the grace clock), `updated_at`; index `(state, next_checkin_at)`); `Checkin` (`id` UUID, `user`, `sent_at`, `channel` `email|in_app`, `responded_at` null, `response` `fine|` blank, `created_at`); `TrusteeConfirmation` (`id` UUID, `user`, `trustee_contact`, `token_hash` unique, `requested_at`, `decision` `pending|confirmed|rejected`, `decided_at`, `ip_address` GenericIP null, `note` Text blank, `created_at`).
- Produces in `inactivity/services.py`: `get_settings(user) -> InactivitySettings` (get_or_create), `evaluate(row, now) -> InactivitySettings`, `respond(user, now) -> InactivitySettings`, `apply_update(row, data: dict, now) -> InactivitySettings`, `decide(confirmation, decision: str, note: str, ip: str | None, now) -> None`.
- Routes: `inactivity` (GET evaluates first; PUT), `checkins/respond` (POST), `trustee/<str:token>` (GET), `trustee/<str:token>/decision` (POST, AllowAny).
- Command: `process_due` prints `released_messages=<n> evaluated=<n>`.

- [ ] **Step 1: Write failing tests (`inactivity/tests/test_engine.py`, `test_api.py`, `test_command.py`; time passed as `now` argument)**

```python
def test_get_creates_defaults(self):
    # GET /inactivity -> {"user_id","enabled": False,"checkin_interval_days": 30,"missed_checkins_limit": 3,
    #                     "grace_period_days": 7,"verify_by_email": True,"trustee_contact_id": None,"state": "active","next_checkin_at": None}
def test_put_enables_and_schedules(self): ...  # enabled true -> next_checkin_at ≈ now + 30 days
def test_put_rejects_foreign_trustee(self): ... # other user's contact -> 400
def test_put_rejects_bad_numbers(self): ...     # checkin_interval_days 0 -> 400
def test_disable_cancels(self): ...             # state "cancelled"

def test_missed_checkin_warns(self):
    # evaluate at next_checkin_at -> Checkin row, email sent (verify_by_email), state "warning", next moved by interval
def test_limit_with_trustee_requests_confirmation(self):
    # 3 missed -> state "pending_trustee", TrusteeConfirmation pending, trustee email has f"{FRONTEND_BASE_URL}/trustee/"
def test_trustee_silent_never_releases(self):
    # pending_trustee, evaluate at now + 365 days -> still "pending_trustee", inactivity messages still "scheduled"
def test_limit_without_trustee_releases_after_grace(self):
    # 3 missed -> warning; evaluate at limit_reached_at + 6 days -> not released; + 7 days -> "released",
    # user.status "deceased_confirmed", inactivity messages "sent"
def test_respond_resets(self):
    # warning state -> POST /checkins/respond -> detail "Check-in recorded. You are marked active.", state "active",
    # open checkins responded_at set, next_checkin_at == now + interval, last_active_at updated
def test_respond_after_release_keeps_released(self): ...

def test_trustee_page_and_confirm(self):
    # GET /trustee/<token> -> {"sender_name","requested_at","status": "pending"}
    # POST decision {"decision": "confirmed"} -> {"detail": "Decision recorded successfully.","decision": "confirmed"}; released
    # second decision -> 400 code "already_decided"; unknown token -> 404
def test_trustee_reject_resets(self): ...       # state "active", next_checkin_at reset

def test_process_due_command(self):
    # one due date message + one enabled user past next_checkin_at -> output contains "released_messages=1" and "evaluated=1"
```

- [ ] **Step 2: Run** `venv/bin/python manage.py test inactivity` — Expected: FAIL.
- [ ] **Step 3: Implement** — timer `OnCalendar=*:0/5` using `app-backend-manage@process_due.service`, same style as the template timers.
- [ ] **Step 4: Run** `venv/bin/python manage.py test` — Expected: OK.

---

### Task 11: Notifications RAML shape, FAQ, business leads

**Files:**
- Modify: `notifications/serializers.py`, `notifications/views.py`, `notifications/urls.py`, `support/models.py`, `support/serializers.py`, `support/views.py`, `support/admin_views.py`, `support/urls.py`, `support/admin.py`, `config/urls.py`
- Create: `notifications/tests/test_raml_notifications.py`, `support/tests/test_raml_support.py`

**Interfaces:**
- Produces: `GET /api/v1/notifications` → array of `{"id","type","title","body","data","read_at","created_at"}` (active, newest first, `id` string). `PATCH /api/v1/notifications/<int:pk>/read` → `{"id","read_at"}`. Template extra routes (unread-count, mark-all-read, clear-all, admin/send, detail) stay.
- Produces: `support.models.BusinessLead` per ERD (`id` UUID, `name`, `company`, `email`, `phone` blank, `message` Text, `status` `new|contacted|closed` default `new`, `created_at`). Note: do not inherit `TimeStampedModel` (its `status` field clashes); use plain `models.Model`.
- Routes: `faq` (GET, AllowAny; published `SupportFAQ` by order → `{"question","answer","category"}` with category name or null), `business-leads` (POST, AllowAny, throttled `business_lead` `5/hour` per IP → 201 detail `"Lead submitted successfully."`), `support/admin/leads` (GET list) and `support/admin/leads/<uuid:pk>` (PATCH status), both `HasPermission` with `support.read` / `support.update`.

- [ ] **Step 1: Write failing tests** — envelope + exact key sets above; other user's notification read → 404; lead missing email → 400; admin leads without permission → 403, with `support.read` → list.
- [ ] **Step 2: Run** `venv/bin/python manage.py test notifications support` — Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `venv/bin/python manage.py test` — Expected: OK.

---

### Task 12: RAML conformance sweep, docs, final checks

**Files:**
- Create: `core/tests/test_raml_routes.py`
- Modify: `README.md`, `.env.example`, `.github/workflows/ci.yml`

**Interfaces:**
- Consumes: everything.

- [ ] **Step 1: Write `test_raml_routes.py`**

Parse `RAML/RAML.yaml` (PyYAML is not a dependency: walk it with a small regex over lines that start with `/` at any indent plus method keys, or hard-code every `(method, path)` pair in the RAML). For each, substitute `{id}`/`{deliveryId}` with a UUID and `{token}` with `"x"`, call with no auth, and assert the response is not 404-from-routing and not 405, and the body has keys `{"status_code","data","error"}`. (A 404 from a missing object is fine; distinguish by asserting `django.urls.resolve` succeeds for the path.)

- [ ] **Step 2: Run** `venv/bin/python manage.py test core.tests.test_raml_routes` — Expected: PASS (fix any route it finds missing).

- [ ] **Step 3: Update docs** — README: Katalog name, setup commands (`seed_rbac`, `seed_legal`, `seed_plans`, `process_due` timer), env vars (`MESSAGE_ENCRYPTION_KEY` with generation command `python -c "import base64,os;print(base64.b64encode(os.urandom(32)).decode())"`), endpoint list pointer to `RAML/RAML.yaml` and `/api/v1/docs`.

- [ ] **Step 4: Full verification**

Run: `venv/bin/ruff check . && venv/bin/mypy . && venv/bin/python manage.py check --deploy --fail-level ERROR && venv/bin/python manage.py makemigrations --check --dry-run && venv/bin/python manage.py test`
Expected: all clean, tests `OK`.

- [ ] **Step 5: Smoke run** — `venv/bin/python manage.py runserver`, then with curl: register → login → create contact → create instant in_app message to a self contact → confirm → `/api/v1/inbox` shows it. Expected: every response in the RAML envelope.
