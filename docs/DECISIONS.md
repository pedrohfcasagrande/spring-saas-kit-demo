# Architecture decisions

Each entry: what we chose, what we rejected, and why. If your context differs, the "why" tells you when to change it.

## 1. Stateless JWT with rotating refresh tokens — not server sessions
**Chose:** 1-hour access token, 30-day refresh token stored in DB (`refresh_tokens`), rotated on every use.
**Rejected:** Spring Session with Redis; opaque tokens validated per request.
**Why:** No Redis to run on a €4 VPS; horizontal scaling stays trivial; the DB-backed refresh token is what makes logout and "revoke all devices" possible, which pure JWT can't do. The cost is that an access token lives for up to 1 hour after revocation — acceptable for most SaaS, change `jwt.expiration-ms` if not.

## 2. Impersonation header instead of per-tenant URLs
**Chose:** `X-Impersonate-User-Id` resolved in `CurrentUser`, with a strict allow-matrix (ADMIN → anyone, COMPANY → its members, others → nobody).
**Rejected:** `/companies/{id}/...` in every route; separate admin API.
**Why:** Support and admin screens reuse the exact same endpoints the user sees, so there is one code path to secure and test. Every impersonated request is written to `audit_log`. The matrix lives in one class; if you need a new rule, it's one `switch` arm.

## 3. Flyway with `ddl-auto: validate` — never `update`
**Chose:** Hand-written SQL migrations; Hibernate only validates.
**Rejected:** `ddl-auto: update`; Liquibase.
**Why:** `update` silently drifts production. Validate catches the drift at boot, and the CI job `backend-schema-smoke` runs the migrations against a real MySQL so H2 quirks never hide a bad migration. Liquibase is fine; Flyway's plain SQL is easier to review in a PR.

## 4. H2 for unit tests, real MySQL in CI
**Chose:** `test` profile on H2 in MySQL mode for speed; one CI job on MySQL 8 for schema validation.
**Rejected:** Testcontainers everywhere.
**Why:** The full suite stays under a minute locally. Testcontainers on every test doubles CI time and needs Docker on every developer machine. The one MySQL job is enough to catch type and naming mismatches.

## 5. Payment providers behind one interface, selected by currency
**Chose:** `PaymentProviderClient` with `createCheckout / changePlan / cancel / reactivate`; `PlanPrice` carries both `currency` and `provider`.
**Rejected:** Stripe only; a payments SaaS (Paddle, Lemon Squeezy).
**Why:** Stripe doesn't serve Brazilian cards and PIX well; Mercado Pago doesn't serve the rest of the world. If you sell in one region, delete the other client — the interface costs nothing. Merchants-of-record take 5–8% and own your customer relationship; worth it only if you can't handle VAT yourself.

## 6. Webhooks are the source of truth for subscription state
**Chose:** Checkout only redirects; the subscription row is created/updated by webhooks, idempotently via `webhook_events(provider, provider_event_id)`.
**Rejected:** Activating the plan on the success redirect.
**Why:** The redirect can be skipped, replayed or faked. Webhooks are signed, retried by the provider, and arrive in any order — both `checkout.session.completed` and `customer.subscription.created` upsert, so order doesn't matter. In `dev` the mock provider activates instantly so you can work offline.

## 7. Feature gates as data, not annotations
**Chose:** `plan_features` rows + `FeatureGateService.check(feature)`; usage counters in `subscription_usage` per month.
**Rejected:** `@RequiresPlan("PRO")` annotations; hard-coded `if (plan == PRO)`.
**Why:** Pricing changes weekly in a young product. With gates as data you add a plan or move a feature between tiers with a migration, not a release. Annotations scatter pricing logic through controllers.

## 8. Company subscription covers members
**Chose:** `MEMBER`s inherit the entitlements of their `COMPANY`'s active subscription; `USER`s hold their own.
**Rejected:** Per-seat billing on every member.
**Why:** B2B buyers want one invoice. Seat counting is enforced by the `maxMembers` feature gate instead. If you need per-seat, Stripe quantity on the subscription item is a small change in `changePlan`.

## 9. Docker Compose + Caddy on a VPS — not Kubernetes, not serverless
**Chose:** One VPS, `docker-compose.prod.yml`, Caddy for TLS, `mysqldump` sidecar for backups.
**Rejected:** ECS/EKS, Cloud Run, managed DB.
**Why:** Until ~10k users this runs on €4–20/month and one person can operate it. Kubernetes triples the surface area for a single-developer product. The compose file is also the documentation of what runs where. Move the DB to a managed service when backups and failover keep you up at night, not before.

## 10. MySQL 8 — not PostgreSQL
**Chose:** MySQL because that's what the origin product ran on and what cheap managed offerings default to in several regions.
**Honest note:** PostgreSQL is the better database for new projects. Porting is a day: the Java code is dialect-neutral; only the 10 migration files need touching.

## 11. Lombok + records
**Chose:** Lombok on entities, Java records for DTOs.
**Why:** Entities need mutability for JPA; DTOs don't. Records give you immutability and `equals` for free and keep controllers readable.

## 12. No module system, one Maven module
**Chose:** Package-by-layer in a single module.
**Rejected:** Hexagonal packages, multi-module Maven.
**Why:** At this size, layers are what every Spring developer expects and navigates fastest. Split when a package exceeds ~40 classes or two teams need independent deploys.
