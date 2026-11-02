# Spring SaaS Kit

**A production-ready multi-tenant SaaS backend you can deploy in 15 minutes** — Spring Boot 3.3 · Java 21 · MySQL 8 · Flyway · Stripe + Mercado Pago.

Built from a real B2B2C product, not from a tutorial. Every piece here has handled real users, real webhooks and real support tickets.

## What's inside

| Area | What you get |
|---|---|
| **Auth** | JWT access + rotating refresh tokens, Google Sign-In, password reset by e-mail, invitation flow with expiry, BCrypt |
| **Multi-tenancy** | `Company → Members (roles) → Users`, scoped queries, safe impersonation (`X-Impersonate-User-Id`) for support/admin |
| **Billing** | Plans with multi-currency prices, subscriptions, feature gates and monthly usage limits, upgrade/downgrade, cancel-at-period-end, reactivate, Stripe Customer Portal |
| **Payments** | Stripe (USD/EUR) and Mercado Pago (BRL) behind one `PaymentProviderClient`; idempotent webhooks stored in `webhook_events`; mock provider for dev |
| **Platform** | Audit log, transactional e-mail, Web Push (VAPID), currency detection, health endpoint |
| **Ops** | Profiles `dev / sandbox / prod / test`, 10 Flyway migrations, GitHub Actions CI with schema smoke test against real MySQL, Docker Compose for prod with Caddy (auto-TLS) and daily DB backups |
| **Docs** | `docs/DECISIONS.md` (why each choice), `docs/BILLING-SETUP.md`, `DEPLOY.md` |

## 15 minutes to first deploy

**0:00 — Run locally**
```bash
git clone <your-copy> my-saas && cd my-saas
docker compose -f docker-compose.dev.yml up -d          # MySQL on :3307
cp backend/src/main/resources/application-local.yml.example backend/src/main/resources/application-local.yml
cd backend && ./mvnw spring-boot:run                     # http://localhost:8081/swagger-ui.html
```
Login with the seeded admin (`ADMIN_EMAIL` / `ADMIN_PASSWORD` from `application-local.yml`). Payments are mocked in `dev`: checkout activates the plan instantly.

**0:05 — Make it yours**
```bash
./scripts/rename.sh io.saaskit com.acme.myapp "My App"   # package, artifactId, {{APP_NAME}}
```
Edit `V005__plans_and_subscriptions.sql`: plan codes, prices, limits.

**0:08 — Stripe**
Create Products/Prices in Stripe, paste the `price_…` ids into `provider_price_id` (see `docs/BILLING-SETUP.md`). Add webhook `https://<domain>/api/webhooks/stripe`.

**0:10 — Deploy**
Follow `DEPLOY.md`: a €4/month VPS, `cp .env.prod.example .env.prod`, fill the secrets, then
```bash
docker compose --env-file .env.prod -f docker-compose.prod.yml up -d --build
```
Caddy fetches the TLS certificate; Flyway runs the migrations; the health check turns green.

**0:15 — Ship your first module**
```bash
./scripts/new-module.sh Invoice          # entity, repository, service, controller, DTO, migration stub, test
```

## Architecture in one picture
```
Client ─► Caddy (TLS) ─► nginx (SPA + /api proxy) ─► Spring Boot ─► MySQL
                                                       │
                                        Stripe / Mercado Pago webhooks
```
Role model: `ADMIN` (platform) · `COMPANY` (tenant owner) · `MEMBER` (staff of a company) · `USER` (end customer). A `COMPANY` subscription covers all its members; `USER`s can hold their own.

## What this kit is not
- Not a framework: it's plain Spring Boot you already know, organised the way a production app ends up after a year.
- Not a UI kit: the Angular frontend (login, pricing, billing, admin) ships separately as an add-on.
- Not multi-database: MySQL 8 only. PostgreSQL port is a day of work and the migrations are the only MySQL-specific part.

## Requirements
Java 21 · Docker · a Stripe or Mercado Pago account for real payments.

## License
Use in unlimited personal and client projects. No resale or redistribution of the kit itself. See `LICENSE`.

## Support
Open an issue in the private repo you get access to. Fixes to the kit ship to all buyers.

---
**This repository contains documentation only.** The full source (backend, frontend, scripts, E2E suite) is delivered as a private GitHub repository to buyers: **[Get the kit on Gumroad](https://GUMROAD_LINK)**
