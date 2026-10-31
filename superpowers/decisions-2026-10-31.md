# Decisions made during the build (2026-10-30 / 2026-10-31)

Every call the builder made on the owner's behalf, in order, with what it costs if wrong.

## Katalog backend (plan 2026-10-30-katalog-backend)

- Ruling: T1 rsync must keep `.superpowers/` and `.impeccable/` lines in .gitignore and keep repo docs/RAML; README.md replaced by template's (rewritten in T12) — template README is better start — cost if wrong: trivial.
- Ruling: T4 keeps a minimal authenticated GET /api/v1/me returning UserSerializer until T5 replaces it — keeps T2 envelope test green — cost if wrong: none.
- Ruling: T2 test_validation_error_envelope asserts `details` is a non-empty dict (not "email" key) because template login serializer field is not `email` until T4 — T4 tests assert email — cost if wrong: none.
- Ruling: keep AbstractUser `is_active`/`date_joined`; `status=suspended` and `deleted_at` both refuse login in T4 (T4 login check: is_active and deleted_at is None and status != suspended) — template/admin code relies on is_active — cost if wrong: one extra column.
- Ruling: keep distinct OTP refusal reasons (no_code vs invalid) despite account-enumeration signal — RAML register already reveals taken emails (400 email), both throttled per IP/email — cost if wrong: enumeration via OTP verify; fix = merge reasons.
- Ruling: welcome notification on register dropped (not in RAML/spec) — cost if wrong: re-add via user_registered receiver.
- Task 4: note: /auth/refresh returns 401 for invalid token (RAML lists only 400) — brief mandated 401; Ruling: keep 401 — Android must treat 401 on refresh as logout, standard — cost if wrong: Android expects 400 only.
- Ruling: deleted-account email `deleted+<id>@deleted.invalid` instead of spec `@invalid.local` — .invalid is the RFC 2606 reserved TLD — cost if wrong: none.
- Ruling: /plans ordered free, monthly, one_time, business (test expects this order; business price 0.00) — cost if wrong: display order only.
- Ruling: on user_deleted, other owners' contacts linked to deleted user get linked_user=None — deleted account must not receive in-app messages — cost if wrong: links lost (re-link on new register by same email).
- Ruling: POST /contacts `relation` optional (RAML lists it required) — accepting more than contract never breaks Android; model allows blank per plan — cost if wrong: none for client.
- Ruling: MESSAGE_ENCRYPTION_KEY falls back to dev key derived from SECRET_KEY only when DEBUG on; DEBUG off + missing/bad key -> ImproperlyConfigured — keeps local/tests zero-setup — cost if wrong: dev data encrypted with derived key.
- Ruling: trigger_type may change to `instant` only while draft (else 400 invalid_state) — avoids stranded scheduled instant messages — cost if wrong: user must pause->? no; user deletes and recreates.
- Ruling: occasion_type emitted null when blank (RAML enum has no "") — cost if wrong: none.
- Ruling: storage quota counts all media rows (uploaded or pending) — reserves quota at upload-url time, blocks parallel over-quota uploads — cost if wrong: abandoned uploads hold quota until deleted.
- Ruling: DATA_UPLOAD_MAX_MEMORY_SIZE from env, default 100 MB, PUT body read into memory — simple; cost if wrong: large videos use RAM per request (switch to streaming later).
- Ruling: soft-deleting a message deletes its media rows + files on commit — recipients lose access to deleted messages anyway; frees quota — cost if wrong: sender cannot undelete media (no undelete exists).
- Ruling: release creates deliveries `queued` in transaction; emails sent after commit, delivery then `sent`/`failed` — no SMTP inside DB transaction — cost if wrong: crash between commit and send leaves `queued` deliveries (process_due could retry later).
- Ruling: email channel + contact without email -> delivery `failed`, last_error "no email" — cost if wrong: none.
- Ruling: inbox/token lookup statuses = sent, delivered, opened (corrects my earlier "sent, opened" — in_app deliveries are `delivered`) — cost if wrong: none.
- Ruling: post-commit email send is synchronous on request path (instant confirm) — simpler, final state visible — cost if wrong: confirm response latency = SMTP time.
- Ruling: SystemLog redacts any URL kwarg named `token` generically (covers T10 trustee links) — cost if wrong: none.
- Ruling: retry_stalled_deliveries(now) built in T9, called only from T10 process_due (not request paths) — carry into T10 dispatch.
- Ruling: released deliveries survive contact soft-deletion (recipient_access_allowed drops live-contact filter) — matches inbox/lookup — cost if wrong: sender cannot revoke a released message by deleting contact.
- Ruling: trustee contact must have email (400 trustee_email_required) — trustee link only goes by email — cost if wrong: users without trustee email must use grace-only mode.
- Ruling: I1 — a check-in counts as missed only after one full interval unanswered; escalation never at the instant a check-in is sent — release is irreversible, err toward living user — cost if wrong: escalation one interval later than spec wording.
- Ruling: I3 — request-path releases bounded (limit 20), per-message savepoints, email deliveries left queued for process_due — cost if wrong: email recipients wait up to 5 min (timer) after release.
- Ruling: I5 — trustee FK PROTECT; deleting current trustee contact -> 400 contact_is_trustee; blanking trustee email -> 400 trustee_email_required — cost if wrong: user must change trustee before deleting contact.
- Ruling: I6 — /notifications list GET-only (RAML has no POST) — cost if wrong: none.
- Ruling: I8 — Docker deploy gets compose `scheduler` service running process_due every 300s — cost if wrong: none.
- Ruling: M4 — all 401 error codes mapped to `unauthorized` per RAML securitySchemes example — cost if wrong: Android branching on specific 401 codes (unlikely).
- Final: parked (load-bearing, surface at finish) — I3 regression: instant-confirm on_commit send vs process_due retry_stalled_deliveries(older_than=0) can double-email with dead first link — Ruling: present to user at finish with fix (row lock + token-hash re-check in _send_emails) — cost if left: duplicate emails, first link dead.
- Final: parked — Sentry still sends Referer header carrying /r/ and /trustee/ tokens — Ruling: present at finish, 2-line fix in _redact_urls — cost if left: tokens in Sentry error events from frontend link pages.
- Final: parked — admin soft-delete runs destructive user_deleted cleanup (media files, sent messages) though admin restore exists — Ruling: present to user (product decision: narrow admin delete vs accept) — cost if left: irreversible data loss on admin delete.
- Ruling: docs/redoc pages get a CDN-allowing CSP (DOCS_CSP) scoped to those two routes — Swagger UI can't run under site CSP — cost if wrong: looser CSP on docs pages only.
- Ruling: R3 — admin soft-delete is reversible: new user_deactivated signal does non-destructive cleanup only (switch off, devices); user_deleted (destructive) only for DELETE /me — admin has restore — cost if wrong: admin-deleted user's media keeps using disk until purged.

## Admin dashboard (plan 2026-10-31-katalog-dashboard)

- Ruling: Behance unreachable -> palette derived from "Future Memory Vault", all tokens in lib/theme/tokens.dart — cost if wrong: one-file palette swap.
- Ruling: Flutter tasks start before backend tasks (backend repo busy with residual fixes); F1 mocks /admin/control/me — cost if wrong: none.
- Ruling: suspend on deceased_confirmed -> 400 invalid_state (extension accepted; dashboard never touches deceased status) — cost if wrong: none.
- Ruling: retry refuses contact without email (400 invalid_state) — avoids repeat undeliverable notifications — cost if wrong: none.
- Ruling: F6 before F5 — F5 waits on B4; F6 uses existing support endpoints — cost if wrong: none.
- Ruling: "Active (not resolved)" ticket filter = open,in_progress,waiting_user; overview link targets it — matches KPI definition — cost if wrong: none.
- Ruling: public support attachments (pre-existing template) left out of this wave — separate ticket, surfaced to user — cost if left: uploaded files readable by path guess.

## Open items

- Support ticket attachments are stored under public MEDIA_ROOT and served at /media without auth (pre-existing template behaviour). Move to PRIVATE_MEDIA_ROOT behind an authorized download view.
- Deferred minor findings are listed in the SDD ledgers' 'minor (deferred)' lines (summarised in the final review triage); none block merge.
