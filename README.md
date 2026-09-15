┌─────────────────┐     ┌─────────────────┐     ┌──────────────────┐
│  Customer Web    │     │   Admin Panel    │     │ Expert (Flutter) │
│  Next.js 15 PWA  │     │   Next.js 15     │     │ Riverpod + Dio   │
└────────┬─────────┘     └────────┬─────────┘     └────────┬─────────┘
         │                        │                         │
         └────────────┬───────────┴─────────────┬───────────┘
                       │        HTTPS / JSON      │
                       ▼                          ▼
              ┌────────────────────────────────────────┐
              │   Nginx (reverse proxy, TLS, gzip)      │
              └────────────────────┬─────────────────────┘
                                   ▼
              ┌────────────────────────────────────────┐
              │        Laravel 13 API (/api/v1)         │
              │  Sanctum · Policies · Form Requests ·   │
              │  API Resources · Events/Listeners ·     │
              │  Service Layer · Jobs · Scheduler       │
              └───────┬─────────────────┬────────────────┘
                      │                 │
             ┌────────▼──────┐  ┌───────▼────────┐
             │ PostgreSQL 16 │  │    Redis 7      │
             │ source of     │  │ cache/queue/     │
             │ truth         │  │ session/rate-limit│
             └───────────────┘  └───────┬─────────┘
                                        ▼
                              ┌───────────────────┐
                              │  Queue Workers      │
                              │  notifications, SMS,│
                              │  payments, matching  │
                              └───────────────────┘

External: Zarinpal (payments) · SMS Provider (Kavenegar-ready, Mock in dev) · FCM (push) · S3-compatible storage
One Laravel backend serves all three clients through a single versioned REST API. No client talks to Postgres/Redis directly. All money logic (commission calc, wallet mutation, payment verification) is server-side only — never trusted from a client payload.
2. Repository / Folder Structure
Monorepo, four apps + shared infra:
khedmatino/
├── backend/                     # Laravel 13 API
│   ├── app/
│   │   ├── Domain/               # bounded contexts, not just MVC buckets
│   │   │   ├── Auth/
│   │   │   ├── Users/            # customers, experts, roles
│   │   │   ├── Catalog/          # categories, services
│   │   │   ├── Requests/         # service_requests lifecycle
│   │   │   ├── Booking/          # bookings, quotes (secondary flow)
│   │   │   ├── Matching/         # scoring engine, availability search
│   │   │   ├── Payments/         # gateway abstraction (Zarinpal/Mock/COD)
│   │   │   ├── Wallet/           # wallet, withdrawals, commission
│   │   │   ├── Reviews/
│   │   │   ├── Chat/
│   │   │   ├── Notifications/
│   │   │   ├── Support/
│   │   │   ├── CMS/
│   │   │   └── Admin/            # audit logs, settings
│   │   ├── Http/
│   │   │   ├── Controllers/Api/V1/{Customer,Expert,Admin}/
│   │   │   ├── Requests/
│   │   │   ├── Resources/
│   │   │   └── Middleware/
│   │   ├── Models/
│   │   ├── Policies/
│   │   ├── Events/  Listeners/  Jobs/  Notifications/
│   │   └── Support/              # StatusMachine, Money value object, etc.
│   ├── database/{migrations,seeders,factories}/
│   ├── routes/api_v1.php
│   ├── tests/{Unit,Feature}/
│   ├── docker/
│   └── .env.example
├── customer-web/                 # Next.js 15 (App Router, RTL, PWA)
├── admin-panel/                  # Next.js 15
├── expert-app/                   # Flutter
├── docs/
│   ├── ARCHITECTURE.md
│   ├── DATABASE.md
│   ├── API.md
│   ├── SECURITY.md
│   ├── DEPLOYMENT.md
│   ├── TESTING.md
│   └── ROADMAP.md
├── docker-compose.yml
└── README.md
Each Domain/<X> holds Services and DTOs for that context; Controllers stay thin. Repository pattern is used only where querying is genuinely complex/swappable (Matching Engine, Reporting) — not blanket CRUD.
3. Key Design Decisions
Direct Booking primary flow: customer opens an expert's public profile (rating, fixed price, distance, availability calendar) and books directly for a fixed-price service. quotes / quote_items tables and the QUOTING / EXPERT_SELECTED states stay in the schema and API so quote-based negotiation can be enabled per-category later without a migration.
Zarinpal is the real payment gateway for MVP, implemented behind a PaymentGateway interface (createPayment, verifyPayment, refundPayment), alongside MockGateway (automated tests) and CashOnDeliveryGateway. This sandbox has no network egress and no live merchant account, so the Zarinpal client code will be complete and correct against Zarinpal's documented REST contract, but the actual callback round-trip must be exercised from your own server with your ZARINPAL_MERCHANT_ID and a real callback domain.
4. Security Model (full detail in SECURITY.md)
Sanctum tokens; separate abilities per guard (customer, expert, admin).
RBAC via roles / permissions / user_roles, enforced with Policies on every resource, not just route middleware.
Every private-data query scoped to the authenticated user's ownership — no IDOR (an expert can never load another expert's requests by guessing an ID).
Form Requests validate + whitelist input (no mass assignment); API Resources control exactly what's serialized per role.
Redis-backed rate limiting on OTP and auth endpoints.
All financial mutations (payments, wallet_transactions, commissions, withdrawal_requests) run inside DB transactions with lockForUpdate on wallet balance rows.
admin_activity_logs + audit_logs populated via model observers on sensitive tables (not manually per-controller, so logging can't be forgotten).
5. Deployment Architecture
docker-compose.yml: app (php-fpm), nginx, postgres, redis, queue (supervised queue:work), scheduler (cron running schedule:run every minute).
Same images across local → staging → production, differing only by .env.
Migrations run as an explicit release step (php artisan migrate --force), never on app boot.
File storage behind the Storage facade: local disk in dev, S3-compatible object storage (e.g. Liara/ArvanCloud, both usable from Iran) in production.
Sentry DSN via .env, no-op when empty.
6. Roadmap
See ROADMAP.md for Phases 0–18. Phase 1 starts with the Laravel skeleton, Docker Compose, and the full migration set from DATABASE.md.
