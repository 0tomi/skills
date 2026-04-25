# Worked Examples

Three reference plans showing how the skill produces output at different sizes and modes. Read whichever matches your case before drafting.

- **Example 1 — Small, sequential.** A focused bug fix / single-feature plan.
- **Example 2 — Medium, sequential.** A multi-layer feature with tests and migration.
- **Example 3 — Medium, orchestrated.** Same kind of scope as example 2, but split across parallel agents.

These are reference shapes, not templates to copy verbatim. Adapt visual choices to your real plan — see `visual-tools.md`.

---

## Example 1 — Small, sequential

**User request:** "Plan adding a daily backup retention limit so old `pg_dump` files get pruned after 14 days."

```markdown
# Implementation Plan

## 1. Scope

### 1.1 Goal
Add a retention limit to the existing pg_dump backup routine so that backup files older than 14 days are deleted automatically after each successful backup. End state: a single new pruning step runs at the end of the existing backup flow, with the retention window configurable via an env var.

### 1.2 Non-goals
- Changing how pg_dump is invoked.
- Adding remote / off-site backup.
- Compression or encryption changes.

### 1.3 Assumptions and constraints
- Backups are written to `%APPDATA%/SuitAPI/backups/` with timestamped names following the existing pattern `suitapi-YYYYMMDD-HHMMSS.dump`.
- The Qt backup orchestrator is the only writer to this folder.
- Retention default of 14 days is acceptable; configurable via `SUITAPI_BACKUP_RETENTION_DAYS`.
- If wrong: if other processes write here, indiscriminate deletion would damage them. Verify before merging.

### 1.4 Affected surface
- `src/backup/backup_manager.cpp` — Modify (add post-backup prune step)
- `src/backup/backup_manager.h` — Modify (declare new method)
- `src/config/env.cpp` — Modify (read retention env var)
- `tests/backup/backup_manager_test.cpp` — Modify (cover prune logic)

### 1.5 Initial risks
- Mis-parsed timestamps in filenames could cause valid backups to be deleted.
- Time zone handling between filename timestamp and "now" must match.

### 1.6 Overall size
**S.** Single module, one new method, no contract changes, deterministic logic.

## 2. Phases

### Phase 1 — Add retention pruning to backup flow

**Goal.** After a successful pg_dump completes, files older than the retention window are deleted from the backup directory.

**Preconditions.** None beyond current main branch.

**Inputs / Outputs (contract).**
- Inputs: existing backup flow exit point in `BackupManager::runBackup()`.
- Outputs: a new method `BackupManager::pruneOldBackups()` invoked at the end of `runBackup()`. New env var `SUITAPI_BACKUP_RETENTION_DAYS` (default 14).

**Tasks.**
1. Add `pruneOldBackups(int retentionDays)` to `BackupManager`, scanning the backup directory, parsing filenames against the existing timestamp pattern, and deleting files older than now − retentionDays.
2. Skip files that don't match the pattern (do not delete unrelated files).
3. Add `BackupConfig::retentionDays()` reading `SUITAPI_BACKUP_RETENTION_DAYS`, defaulting to 14, validating ≥1.
4. Call `pruneOldBackups()` at the end of `runBackup()`, only on success.
5. Log every deletion (path + age in days) and any parse-skip with the file name.

**Touched files.** All four files from §1.4.

**Validation.**
- Compiles on the standard CI config.
- Unit test: 5 fake files, ages [1, 7, 14, 15, 30] days, retention 14 → only the 15- and 30-day files deleted.
- Unit test: file with non-matching name is never deleted.
- Unit test: invalid env value falls back to 14 with a logged warning.

**Risks.**
- Filename parsing edge cases (DST, leading zeros). Mitigate with explicit test cases.
- Concurrent backup running while prune executes. Out of scope; runBackup is the only caller and is serialized.

**Complexity.** S — local change, well-scoped tests.

**Result.** Backups self-prune to a 14-day rolling window after each successful run.

## 3. Execution guidance

### 3.1 Advance rule
Single phase — N/A.

### 3.2 Per-iteration minimums
- Code compiles.
- Existing backup tests still pass.
- New prune tests pass.

### 3.3 Correction policy
If filename parsing turns out to be more variable than assumed, stop deletion entirely (return early in `pruneOldBackups`) and surface a TODO before re-attempting.
```

**What to notice.** No tables for sections that didn't need them. No Mermaid. Phase has all required fields. Sizing matches reality (one S phase, S overall).

---

## Example 2 — Medium, sequential

**User request:** "Plan adding email-based magic-link auth to the demo tenant flow."

```markdown
# Implementation Plan

## 1. Scope

### 1.1 Goal
Allow prospective customers to access an isolated demo tenant via a magic link sent by email, instead of the current shared password. End state: a public sign-up form generates a single-use, time-limited token, the token authenticates into a freshly-provisioned tenant schema, and expired tokens trigger cleanup of their tenant.

### 1.2 Non-goals
- Replacing auth in the production multi-tenant app.
- Adding social login.
- Changing the tenancy-for-laravel version or the schema-per-tenant model.

### 1.3 Assumptions and constraints
- `tenancyforlaravel` v3 is the active multi-tenancy package.
- Email delivery uses the existing mailer config; no new transport.
- Token TTL: 24 hours by default, configurable via env. **If wrong:** changing this later requires a migration on the tokens table and is non-trivial.
- Tenant lifetime equals token lifetime: when the token expires, the tenant is dropped.

### 1.4 Affected surface
| File | State | Reason |
|---|---|---|
| `app/Http/Controllers/DemoSignupController.php` | Create | Public sign-up endpoint |
| `app/Http/Controllers/DemoAuthController.php` | Create | Token verification + login |
| `app/Models/DemoToken.php` | Create | Token model |
| `database/migrations/2026_xx_create_demo_tokens.php` | Create | Tokens table |
| `app/Mail/DemoMagicLink.php` | Create | Mailable |
| `resources/views/emails/demo-magic-link.blade.php` | Create | Email template |
| `app/Console/Commands/PruneExpiredDemos.php` | Create | Scheduled cleanup |
| `app/Console/Kernel.php` | Modify | Register schedule |
| `routes/web.php` | Modify | Public routes |
| `config/demo.php` | Create | TTL, limits |
| `tests/Feature/DemoSignupTest.php` | Create | E2E coverage |

### 1.5 Initial risks
- Cleanup of an expired tenant must drop the schema reliably; partial drops leave orphaned data.
- Sign-up endpoint is public → abuse vector. Needs rate limiting.
- Mail send failures must not leave dangling tokens or tenants.

### 1.6 Overall size
**M.** Touches HTTP, persistence, mail, scheduling, and the multi-tenancy lifecycle. Six new files, two modified, one migration.

## 2. Phases

### Phase 1 — Token model, table, and config

**Goal.** A persisted, queryable demo-token entity with TTL and single-use semantics.

**Preconditions.** None.

**Inputs / Outputs (contract).**
- Inputs: existing tenancy-for-laravel tenant identifier conventions.
- Outputs: `DemoToken` model with fields `[id, email, token_hash, tenant_id, expires_at, used_at, created_at]`. Migration applied. `config/demo.php` exposes `ttl_hours` (default 24) and `max_active` (default 50).

**Tasks.**
1. Generate migration creating `demo_tokens` with the fields above, indexes on `token_hash` and `expires_at`.
2. Create `DemoToken` model with scopes `active()` and `expired()`.
3. Create `config/demo.php` with `ttl_hours` and `max_active`.
4. Run migration locally; verify schema.

**Touched files.** Migration, `DemoToken.php`, `config/demo.php`.

**Validation.**
- Migration applies and rolls back cleanly.
- Model unit test: `active()` excludes expired and used tokens.
- Config values readable via `config('demo.ttl_hours')`.

**Risks.** Index choices — verify query plans on `expires_at` once volume is meaningful. Out of scope at this size.

**Complexity.** S — straightforward Laravel scaffolding.

**Result.** Tokens can be created and queried; the rest of the flow has a foundation.

### Phase 2 — Sign-up endpoint and magic-link mail

**Goal.** A user submits an email, receives a magic link, and a tenant is provisioned tied to that token.

**Preconditions.** Phase 1 merged.

**Inputs / Outputs (contract).**
- Inputs: `DemoToken` model from Phase 1.
- Outputs: `POST /demo/signup` accepting `{ email }`, creating a new tenant via the tenancy package, persisting a `DemoToken` for that tenant, and dispatching `DemoMagicLink` mail. Returns `204` on success or `429` if rate-limited.

**Tasks.**
1. Create `DemoSignupController@store`. Validate email. Reject if active token count for that email exceeds `max_active`.
2. Provision a new tenant (schema-per-tenant via tenancy-for-laravel).
3. Create a `DemoToken` with hashed token, `expires_at = now + ttl_hours`, link to tenant id.
4. Build and dispatch `DemoMagicLink` mailable carrying the unhashed token in a verification URL.
5. Apply throttle middleware: 5 requests per IP per hour.
6. Register the route in `routes/web.php`.

**Touched files.** Controller, mailable, blade template, `routes/web.php`.

**Validation.**
- Feature test: valid email → tenant created, token persisted, mail dispatched (assert via `Mail::fake()`).
- Feature test: 6th request from same IP within an hour → 429.
- Feature test: tenant schema actually exists in PostgreSQL after sign-up.
- No new lint errors.

**Risks.** Tenant provisioning fails partway → orphaned schema. Mitigate by wrapping in try/catch and rolling back tenant creation on failure before persisting the token.

**Complexity.** M — touches tenancy lifecycle, mail, rate limiting, and validation in one phase.

**Result.** A real magic link arrives in the user's inbox; the tenant exists and is bound to the token.

### Phase 3 — Token verification and login into tenant

**Goal.** Clicking the link logs the user into their demo tenant for the token's TTL.

**Preconditions.** Phases 1–2 merged.

**Inputs / Outputs (contract).**
- Inputs: `DemoToken` from Phase 1, tenant from Phase 2.
- Outputs: `GET /demo/auth/{token}` route. On valid token: marks `used_at`, switches tenancy context, logs the user in as the demo user inside the tenant, and redirects to the tenant home. Invalid or expired → friendly error page.

**Tasks.**
1. Create `DemoAuthController@verify`.
2. Look up token by hash. Reject if not found, expired, or already used.
3. Switch tenancy context to the bound tenant.
4. Authenticate the demo user (pre-seeded fixed user inside the tenant fixtures).
5. Mark `used_at = now`.
6. Redirect to the tenant's landing page.

**Touched files.** `DemoAuthController.php`, `routes/web.php`.

**Validation.**
- Feature test: valid unused token → 302 to tenant landing, session has demo user.
- Feature test: expired token → 410 with friendly view.
- Feature test: already-used token → 410.
- Manual smoke: end-to-end signup → click link → land in tenant.

**Risks.** Tenancy context switch failing leaves the user logged out with no clear feedback. Mitigate by wrapping the switch in a try/catch and rendering the friendly error view.

**Complexity.** M — interaction with tenancy package and auth in the same flow.

**Result.** The full happy path works end-to-end for a single user.

### Phase 4 — Expired-tenant cleanup

**Goal.** Expired tokens trigger deletion of their tenant schema and the token itself.

**Preconditions.** Phases 1–3 merged.

**Inputs / Outputs (contract).**
- Inputs: expired `DemoToken` rows.
- Outputs: `php artisan demo:prune` command, scheduled hourly via `Console\Kernel`.

**Tasks.**
1. Create `PruneExpiredDemos` command. Iterate expired tokens.
2. For each: drop the tenant schema via the tenancy package's tenant deletion API, then delete the token row.
3. Log every deletion with token id, tenant id, and reason (expired / unused).
4. Register hourly schedule in `Console\Kernel`.
5. Make the command idempotent: if tenant already gone, just delete the token.

**Touched files.** Command, `Console\Kernel.php`.

**Validation.**
- Unit test: expired token with existing tenant → both gone after run.
- Unit test: expired token with already-missing tenant → token gone, no error.
- Unit test: unexpired token → untouched.
- Manual: artificially expire a token and run the command.

**Risks.** Dropping a tenant schema is destructive. Verify in staging before scheduling in production.

**Complexity.** M — lifecycle work plus scheduling.

**Result.** Demo tenants are bounded in lifetime; the system does not accumulate dead schemas.

## 3. Execution guidance

### 3.1 Advance rule
Standard sequential. Each phase merges before the next starts.

### 3.2 Per-iteration minimums
- All existing tests pass.
- New tests for the phase pass.
- Lint clean.
- For phases 2–3, manual smoke through the new path.

### 3.3 Correction policy
If tenant provisioning turns out to be slower or less reliable than assumed (Phase 2), stop and re-plan: a queued / async sign-up may be needed instead of synchronous, which changes the contract for Phase 3.
```

**What to notice.** Mixed visual tools — a table where it earned its place (§1.4), prose elsewhere. No Mermaid, because nothing is graph-shaped here. Per-phase complexity, not just overall.

---

## Example 3 — Medium, orchestrated

**User request:** "Plan the same demo magic-link flow, but I want to orchestrate this across multiple agents in parallel."

The §1 Scope is identical to Example 2 — skipping it for brevity. The differences appear in §2 (per-phase additions) and §4 (orchestration section).

### Per-phase additions

Each phase from Example 2 gains three new fields after `Complexity`:

```
Phase 1 — Token model, table, and config
  ...
  Complexity. S
  Parallel group. G1
  Depends on. none
  Suggested agent profile. light — mechanical scaffolding, no architectural decisions

Phase 2 — Sign-up endpoint and magic-link mail
  ...
  Complexity. M
  Parallel group. G2
  Depends on. Phase 1
  Suggested agent profile. standard — touches mail and tenancy, but contract is fixed

Phase 3 — Token verification and login into tenant
  ...
  Complexity. M
  Parallel group. G2
  Depends on. Phase 1
  Suggested agent profile. standard — independent of Phase 2; both consume Phase 1's contract only

Phase 4 — Expired-tenant cleanup
  ...
  Complexity. M
  Parallel group. G3
  Depends on. Phase 1
  Suggested agent profile. heavy — destructive ops, requires careful idempotency reasoning
```

### §4 Orchestration

```markdown
## 4. Orchestration

### 4.1 Dependency graph

Phase 1 → Phase 2
Phase 1 → Phase 3
Phase 1 → Phase 4
Phase 2 + Phase 3 → integration smoke test (covered in §4.4)

### 4.2 Parallel groups

| Group | Phases | Rationale |
|---|---|---|
| G1 | Phase 1 | Foundation — defines the model and config every other phase consumes |
| G2 | Phase 2, Phase 3 | Independent — different controllers, different routes, both consume Phase 1's contract only |
| G3 | Phase 4 | Independent of G2 in code; depends only on Phase 1's model. Can run alongside G2 |

### 4.3 Inter-phase contracts

Phase 1 → Phases 2, 3, 4
  Publishes:
    - DemoToken model with fields [id, email, token_hash, tenant_id, expires_at, used_at]
    - Scopes: active(), expired()
    - config('demo.ttl_hours'), config('demo.max_active')
    - Migration 2026_xx_create_demo_tokens applied
  Downstream must consume from this surface only — do not extend the model
  in parallel phases.

Phase 2 → integration
  Publishes:
    - POST /demo/signup contract: { email } → 204 | 429
    - Side effect: DemoToken row created, DemoMagicLink mail dispatched

Phase 3 → integration
  Publishes:
    - GET /demo/auth/{token} contract: 302 to tenant landing | 410 with error view
    - Side effect: DemoToken.used_at set

### 4.4 Merge criteria

Integration check after G2 completes (before G3 is considered done):
  - End-to-end test: POST /demo/signup with a real email mock, capture the
    dispatched magic link URL, follow it, assert redirect to tenant landing.
  - This test exists in tests/Feature/DemoFlowIntegrationTest.php.
  - Failure → branches do not fit; reconcile by re-reading the contracts in
    §4.3 and identifying which side drifted.

### 4.5 Orchestrator notes

- Phase 1 must be merged before any G2/G3 agent starts. Do not let G2 or G3
  agents see Phase 1 as a draft — they will design against unstable shapes.
- Phase 4 (cleanup) is destructive. Run its tests in an isolated DB. Do not
  retry blindly on failure; surface for review.
- Phase 2 and Phase 3 should run in fresh agent contexts. They share no code
  beyond the Phase 1 model; cross-pollinated context tends to produce
  unjustified shared abstractions.
- After G2 completes, hand the integration test (§4.4) to a separate
  verification agent rather than reusing the implementing agents.
```

**What to notice.**
- The dependency graph here is simple enough that adjacency-list form is more readable than Mermaid. The author chose accordingly.
- Phase 4 is *not* in G2 because it's destructive and benefits from isolation, even though it's technically independent of G2 in code terms. Agent profile (`heavy`) reflects this.
- Contracts are tight — every consumer knows exactly what to read.
- Merge criteria specify a real test, not a vague "make sure things integrate".
