# Multi-Tenant Real-Time Messaging SaaS Platform
## End-to-End Technical Documentation

**Project Type:** Multi-Tenant Real-Time Backend-as-a-Service (Pusher-Compatible)
**Tech Stack:** Next.js, Node.js (NestJS), MySQL, Soketi (WebSocket Server)
**Document Purpose:** Complete development guide covering architecture, features, flow, development steps, and expected outputs.

---

## Table of Contents

1. Project Overview
2. Goals & Objectives
3. Tech Stack Breakdown
4. High-Level Architecture
5. Monorepo Structure
6. Multi-Tenancy Model
7. Database Design (MySQL)
8. Authentication & Authorization Flow
9. App / Tenant Lifecycle
10. Backend (NestJS API) Development Steps
11. WebSocket Server (Soketi) Setup
12. Pusher-Compatible Protocol Implementation
13. Admin Dashboard (Next.js) Development Steps
14. Feature-by-Feature Breakdown
15. API Endpoint Reference
16. Rate Limiting Strategy
17. Usage Analytics System
18. Webhooks System
19. Billing Readiness
20. Security Considerations
21. Development Environment Setup
22. Testing Strategy
23. Deployment Steps
24. Expected Outputs Per Phase
25. Project Roadmap / Milestones
26. Appendix: Sample Configs & Requests

---

## 1. Project Overview

This project is a **Software-as-a-Service (SaaS) real-time communication platform**, functionally similar to Pusher.com, Ably, or Soketi Cloud. It allows multiple customers (tenants) to register on the platform, create isolated "Apps," and use those Apps to power real-time features such as chat, notifications, order tracking, live dashboards, and payment status updates inside their own products.

Each tenant operates independently. A tenant can create multiple Apps, and each App can have multiple channels representing different real-time use cases. For example:

```
Customer A
   App ID
      ├── chat
      ├── orders
      └── notifications

Customer B
   App ID
      ├── chat
      ├── payment
      └── support

Customer C
   App ID
      ├── dashboard
      └── live_tracking
```

The platform gives every customer a **self-service Admin Dashboard** where they can create Apps, generate App ID/Key/Secret credentials, monitor real-time connections, view channel-level analytics, configure webhooks, rotate secrets, and track usage against their plan limits.

Internally, the platform is composed of three cooperating services:

```
apps/
 ├── admin-dashboard   (Next.js)  -> Tenant-facing UI + Super Admin UI
 ├── api               (NestJS)   -> REST API, Auth, Tenant/App management, Analytics
 └── websocket         (Soketi)   -> Pusher-protocol compatible WebSocket server
```

---

## 2. Goals & Objectives

The primary goal is to build a **production-grade, multi-tenant, Pusher-compatible real-time backend service** that can be white-labeled and sold as a SaaS product.

### Core Objectives

- Provide full **tenant isolation** at the data, credential, and channel level.
- Support **unlimited Apps per tenant**, each with independently scoped channels.
- Be **drop-in compatible** with the Pusher Channels client SDK protocol, so customers can integrate using existing Pusher JS/mobile libraries by only changing the host/cluster/key.
- Provide an **Admin Dashboard** for both tenants (customers) and platform super-admins.
- Track **usage metrics** (connections, messages, channels, bandwidth) per App for billing and rate-limiting.
- Offer **webhooks** so tenants can react to server-side events (channel occupied/vacated, member added/removed, client events).
- Be **billing-ready**, meaning usage data maps cleanly onto subscription tiers (e.g., Stripe metered billing).
- Ensure the system is **secure**, with signed requests, JWT-based dashboard auth, and per-App secret-based WebSocket auth channels (private/presence channels).

### Non-Goals (Out of Scope for v1)

- Building a full billing/payment engine (only "billing ready" hooks are built; actual Stripe integration is a later phase).
- Building native mobile SDKs (only documenting compatibility with existing Pusher SDKs).
- Multi-region WebSocket clustering (v1 assumes a single-region deployment with horizontal scaling behind a load balancer).

---

## 3. Tech Stack Breakdown

| Layer | Technology | Purpose |
|---|---|---|
| Frontend (Admin Dashboard) | Next.js 14 (App Router) | Tenant + Super Admin UI, SSR pages, dashboard widgets |
| Styling | Tailwind CSS + shadcn/ui | Consistent, themeable component system |
| Backend API | Node.js + NestJS | REST API, business logic, auth, tenant/app management |
| ORM | Prisma (MySQL connector) | Type-safe database access, migrations |
| Database | MySQL 8 | Primary relational data store |
| Cache / Rate Limit Store | Redis | Rate limiting counters, session cache, pub/sub for analytics |
| Real-Time Server | Soketi (Pusher-protocol compatible, Node-based) | WebSocket connections, channel broadcasting |
| Auth | JWT (access + refresh tokens) | Dashboard session auth |
| Channel Auth | HMAC-SHA256 signed auth (Pusher-style) | Private/presence channel authorization |
| Queue (optional, for webhooks) | BullMQ (Redis-backed) | Reliable webhook delivery with retries |
| Deployment | Native Node.js processes + PM2/systemd, Nginx reverse proxy | Process supervision, horizontal scaling without containers |
| Process Manager | PM2 (cluster mode) | Runs and supervises all Node processes directly on the server — no containers |
| Monitoring | Prometheus + Grafana (optional) | Metrics on connections, throughput, errors |

---

## 4. High-Level Architecture

```
                                Admin Dashboard (Next.js)
                                            │
                    ┌───────────────────────┼───────────────────────┐
                    │                       │                       │
              Tenant Portal          Super Admin Portal        Public Docs/Landing
                    │                       │
                    └───────────┬───────────┘
                                │  (REST calls, JWT auth)
                                ▼
                        NestJS API Gateway
                    ┌───────────┼───────────┐
                    │           │           │
              Auth Module  Tenant Module  Analytics Module
                    │           │           │
                    └───────────┼───────────┘
                                │
                             MySQL 8
                                │
                    ┌───────────┴───────────┐
                    │                       │
             Redis (cache/RL)        Soketi WebSocket Server
                                            │
                ┌───────────────────────────┼───────────────────────┐
                │                           │                       │
          Customer A Clients          Customer B Clients      Customer C Clients
        (chat, orders, notif.)      (chat, payment, support)  (dashboard, tracking)
```

**Data flow summary:**

1. A tenant logs into the Admin Dashboard (Next.js) → authenticates via NestJS `/auth/login` → receives JWT.
2. Tenant creates an App → NestJS generates `app_id`, `key`, `secret` → stored in MySQL.
3. Tenant's product (their own website/app) uses the Pusher JS SDK, pointing `key` and `cluster/host` at our Soketi server.
4. End-users' browsers/apps open WebSocket connections to Soketi, scoped to that `app_id`/`key`.
5. Private/presence channel subscriptions are authorized via a signed request to NestJS (`/pusher/auth`), which validates using the App's `secret`.
6. Every connection/message event is captured (via Soketi webhooks or Redis pub/sub) and written to MySQL usage tables, aggregated for the Analytics Dashboard.
7. Configured Webhooks fire on channel/member events, delivered via a BullMQ queue with retry/backoff.

---

## 5. Monorepo Structure

The project uses a **Turborepo/Nx-style monorepo** so the three apps and shared packages can be developed together.

```
realtime-saas/
├── apps/
│   ├── admin-dashboard/          # Next.js app
│   │   ├── app/
│   │   │   ├── (auth)/login
│   │   │   ├── (auth)/register
│   │   │   ├── dashboard/
│   │   │   │   ├── apps/
│   │   │   │   ├── apps/[appId]/
│   │   │   │   ├── apps/[appId]/keys/
│   │   │   │   ├── apps/[appId]/analytics/
│   │   │   │   ├── apps/[appId]/webhooks/
│   │   │   │   ├── apps/[appId]/connections/
│   │   │   │   ├── billing/
│   │   │   │   └── settings/
│   │   │   └── admin/            # Super admin section
│   │   ├── components/
│   │   ├── lib/
│   │   └── package.json
│   │
│   ├── api/                       # NestJS app
│   │   ├── src/
│   │   │   ├── auth/
│   │   │   ├── tenants/
│   │   │   ├── apps/
│   │   │   ├── channels/
│   │   │   ├── analytics/
│   │   │   ├── webhooks/
│   │   │   ├── billing/
│   │   │   ├── rate-limit/
│   │   │   ├── common/ (guards, interceptors, decorators)
│   │   │   └── main.ts
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   └── migrations/
│   │   └── package.json
│   │
│   └── websocket/                 # Soketi configuration + custom app-provider
│       ├── config/
│       │   └── soketi.config.js
│       ├── app-provider/           # Custom DB-backed app provider (dynamic apps)
│       └── package.json
│
├── packages/
│   ├── shared-types/               # Shared TS interfaces (App, Tenant, Channel, etc.)
│   ├── ui/                         # Shared React components
│   └── config/                     # Shared ESLint/TS config
│
├── ecosystem.config.js        # PM2 process definitions for api/websocket/admin-dashboard
├── turbo.json
└── package.json
```

---

## 6. Multi-Tenancy Model

Multi-tenancy is the foundational architectural decision of this platform. The model chosen is **shared database, shared schema, tenant-scoped rows** (as opposed to database-per-tenant), because it is simpler to operate at small-to-mid scale and still supports strict isolation through application-level scoping and row-level foreign keys.

### Tenancy Hierarchy

```
Tenant (Customer / Organization account)
   └── Users (people who log into the dashboard under that tenant)
   └── Apps (1..N per tenant)
         └── App Credentials (app_id, key, secret)
         └── Channels (created implicitly when clients subscribe)
               └── Channel Type: public | private | presence
         └── Webhooks
         └── Usage Records
         └── Rate Limit Policy
```

### Isolation Rules

- Every table that stores App-scoped data has a `tenant_id` and `app_id` foreign key.
- All NestJS queries pass through a **TenantScopeGuard** that injects the authenticated user's `tenant_id` into every query — a tenant can never read another tenant's rows, even by guessing an ID.
- Soketi is configured with a **custom app provider** that looks up App credentials dynamically from MySQL (instead of Soketi's default static JSON config), so each App's `key`/`secret` pair is validated against the live tenant database.
- Channel names are automatically namespaced internally as `app_{app_id}.{channel_name}` to prevent cross-tenant channel collisions, while the external-facing channel name shown to the tenant's client SDK remains clean (e.g., `chat`, `orders`).

---

## 7. Database Design (MySQL)

### 7.1 Entity Relationship Summary

```
tenants ───< users
tenants ───< apps ───< app_keys
apps ───< channels ───< channel_stats
apps ───< webhooks ───< webhook_deliveries
apps ───< usage_daily
apps ───< rate_limit_policies
apps ───< connections (live/ephemeral, mirrored to Redis)
tenants ───< billing_accounts ───< invoices
```

### 7.2 Core Tables

**`tenants`**
| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK | |
| name | VARCHAR(255) | Company/customer name |
| slug | VARCHAR(100) | Unique, URL-safe identifier |
| plan | ENUM('free','starter','pro','enterprise') | |
| status | ENUM('active','suspended','trial') | |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |

**`users`**
| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK | |
| tenant_id | BIGINT FK → tenants.id | |
| email | VARCHAR(255) UNIQUE | |
| password_hash | VARCHAR(255) | bcrypt hashed |
| role | ENUM('owner','admin','member','super_admin') | |
| last_login_at | TIMESTAMP NULL | |
| created_at | TIMESTAMP | |

**`apps`**
| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK | |
| tenant_id | BIGINT FK → tenants.id | |
| app_id | VARCHAR(32) UNIQUE | Public, e.g. `app_9f8a2b` |
| name | VARCHAR(255) | Human readable label |
| cluster | VARCHAR(50) | Deployment region tag, e.g. `mt1` |
| enable_client_events | BOOLEAN | Allow client-to-client events |
| max_connections | INT | Plan-based cap |
| max_message_size_kb | INT | |
| status | ENUM('active','disabled') | |
| created_at | TIMESTAMP | |

**`app_keys`**
| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK | |
| app_id | BIGINT FK → apps.id | |
| key | VARCHAR(64) UNIQUE | Public key used by client SDK |
| secret | VARCHAR(128) | Encrypted at rest (AES-256) |
| secret_last_rotated_at | TIMESTAMP | |
| is_active | BOOLEAN | Supports key rotation with grace period |
| created_at | TIMESTAMP | |

**`channels`**
| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK | |
| app_id | BIGINT FK → apps.id | |
| name | VARCHAR(200) | e.g. `chat`, `private-orders`, `presence-support` |
| type | ENUM('public','private','presence') | |
| first_seen_at | TIMESTAMP | |
| last_active_at | TIMESTAMP | |

**`channel_stats`** (rolled up periodically from Redis)
| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK | |
| channel_id | BIGINT FK → channels.id | |
| date | DATE | |
| peak_connections | INT | |
| messages_sent | INT | |
| bytes_transferred | BIGINT | |

**`usage_daily`**
| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK | |
| app_id | BIGINT FK → apps.id | |
| date | DATE | |
| connections_count | INT | |
| messages_count | INT | |
| peak_concurrent_connections | INT | |
| bandwidth_bytes | BIGINT | |

**`webhooks`**
| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK | |
| app_id | BIGINT FK → apps.id | |
| url | VARCHAR(500) | Destination endpoint |
| events | JSON | e.g. `["channel_occupied","member_added"]` |
| secret | VARCHAR(128) | Used to sign outgoing payloads |
| is_active | BOOLEAN | |
| created_at | TIMESTAMP | |

**`webhook_deliveries`**
| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK | |
| webhook_id | BIGINT FK → webhooks.id | |
| event_type | VARCHAR(100) | |
| payload | JSON | |
| response_status | INT | |
| attempt_count | INT | |
| delivered_at | TIMESTAMP NULL | |
| created_at | TIMESTAMP | |

**`rate_limit_policies`**
| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK | |
| app_id | BIGINT FK → apps.id | |
| max_messages_per_second | INT | |
| max_connections_per_minute | INT | |
| max_api_requests_per_minute | INT | |

**`billing_accounts`** (billing-ready scaffolding)
| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK | |
| tenant_id | BIGINT FK → tenants.id | |
| stripe_customer_id | VARCHAR(100) NULL | |
| plan | VARCHAR(50) | |
| billing_cycle_start | DATE | |

---

## 8. Authentication & Authorization Flow

There are **two distinct authentication systems** in this platform, and they must not be confused:

1. **Dashboard Authentication (JWT)** — for humans logging into the Next.js Admin Dashboard.
2. **Channel Authentication (HMAC signature)** — for the tenant's *own end-users* subscribing to private/presence WebSocket channels, following the Pusher auth protocol.

### 8.1 Dashboard JWT Flow

```
1. User submits email/password → POST /auth/login
2. NestJS validates password hash (bcrypt.compare)
3. NestJS issues:
     - access_token  (JWT, 15 min expiry, contains tenant_id, user_id, role)
     - refresh_token (JWT, 7 day expiry, httpOnly cookie)
4. Dashboard stores access_token in memory (not localStorage) to reduce XSS risk
5. Every API request → Authorization: Bearer <access_token>
6. NestJS AuthGuard verifies signature + expiry
7. TenantScopeGuard extracts tenant_id from token payload and scopes DB queries
8. On access_token expiry → POST /auth/refresh using httpOnly refresh cookie
9. Logout → refresh token is blacklisted (Redis set) and cookie cleared
```

### 8.2 Channel (Pusher-Style) Auth Flow

This is how a tenant's own end-users authenticate to subscribe to `private-*` or `presence-*` channels:

```
1. End-user's browser (running tenant's product) opens WebSocket to Soketi using App `key`
2. Client SDK requests to subscribe to `private-orders-55`
3. Client SDK calls tenant's own backend authorize endpoint (tenant-hosted), e.g. POST /pusher/auth
4. That tenant backend calls OUR platform's signing helper (or uses our SDK) with:
     socket_id + channel_name + app secret
5. Signature = HMAC_SHA256(secret, `${socket_id}:${channel_name}`)
6. Response: { auth: "key:signature" } sent back to client SDK
7. Client SDK forwards this auth string to Soketi over the existing socket
8. Soketi validates the signature against the App's secret (fetched via custom DB app-provider)
9. If valid → subscription confirmed, channel_id row created/updated in MySQL if new
```

This mirrors the exact Pusher Channels protocol, which is why existing Pusher SDKs work unmodified against our Soketi server.

---

## 9. App / Tenant Lifecycle

### 9.1 Tenant Signup Flow

```
1. Visit /register on Admin Dashboard
2. Fill company name, email, password
3. POST /auth/register
4. NestJS creates tenant row (plan='free', status='trial')
5. Creates first user row with role='owner'
6. Sends verification email (optional, via queued job)
7. Auto-login → redirect to /dashboard (empty state: "Create your first App")
```

### 9.2 App Creation Flow

```
1. Dashboard: Apps → "Create New App"
2. Form: App Name, Cluster/Region, Enable Client Events (toggle)
3. POST /apps { name, cluster, enable_client_events }
4. NestJS:
     a. Generates app_id  = `app_` + nanoid(10)
     b. Generates key     = nanoid(20)
     c. Generates secret  = nanoid(40), encrypted before storage
     d. Inserts into apps + app_keys tables
     e. Creates default rate_limit_policy row based on tenant.plan
5. Response returns { app_id, key, secret } — secret shown ONCE in a modal
     with a "copy & store safely" warning, matching Pusher/Stripe UX patterns
6. Dashboard redirects to /dashboard/apps/[appId] overview page
```

### 9.3 Secret Rotation Flow

```
1. Dashboard: App → Keys → "Rotate Secret"
2. Confirmation modal: "This will invalidate the old secret in 24h. Continue?"
3. POST /apps/:appId/rotate-secret
4. NestJS:
     a. Marks current app_keys row is_active = false, but keeps valid for grace_period (24h)
     b. Inserts new app_keys row (new secret, is_active = true)
     c. Soketi app-provider now returns BOTH secrets during grace period
5. After grace period, a scheduled job (cron) purges the expired secret row
```

### 9.4 App Deletion Flow

```
1. Dashboard: App → Settings → "Delete App" (type app name to confirm)
2. DELETE /apps/:appId
3. NestJS soft-deletes: apps.status = 'disabled', schedules hard-delete job in 30 days
4. Soketi app-provider immediately stops accepting new connections for that app_id
5. Existing open sockets for that app are force-disconnected via Soketi admin API
```

---

## 10. Backend (NestJS API) Development Steps

Below is the **step-by-step build order** for the API service.

### Step 1 — Project bootstrap
```bash
npx @nestjs/cli new api
cd api
npm install @nestjs/config @nestjs/jwt @nestjs/passport passport passport-jwt
npm install @prisma/client bcrypt nanoid ioredis
npm install -D prisma
npx prisma init --datasource-provider mysql
```

### Step 2 — Define Prisma schema
- Create all tables listed in Section 7 inside `prisma/schema.prisma`.
- Run `npx prisma migrate dev --name init`.

### Step 3 — Core modules scaffold
```bash
nest g module auth
nest g module tenants
nest g module apps
nest g module channels
nest g module analytics
nest g module webhooks
nest g module billing
nest g module rate-limit
```

### Step 4 — Auth Module
- Implement `AuthService.register()`, `login()`, `refresh()`, `logout()`.
- Implement `JwtStrategy` (validates access token, attaches `req.user`).
- Implement `RolesGuard` (owner/admin/member/super_admin).
- Implement `TenantScopeGuard` (injects `tenant_id` filter automatically).

### Step 5 — Tenants Module
- CRUD endpoints for tenant profile/settings (super-admin only for cross-tenant listing).
- Endpoint: `GET /tenants/me` returns current tenant profile + plan.

### Step 6 — Apps Module
- `POST /apps` — create app (Section 9.2 logic).
- `GET /apps` — list apps for current tenant.
- `GET /apps/:appId` — app detail + live connection count (pulled from Redis).
- `PATCH /apps/:appId` — update settings (client events toggle, limits).
- `DELETE /apps/:appId` — soft delete (Section 9.4).
- `POST /apps/:appId/rotate-secret` — Section 9.3 logic.

### Step 7 — Channels Module
- `GET /apps/:appId/channels` — list channels with last-active timestamps.
- `GET /apps/:appId/channels/:channelName/stats` — historical stats.
- Internal listener that consumes Soketi webhook events (`channel_occupied`, `channel_vacated`, `member_added`, `member_removed`) and upserts `channels` + `channel_stats` rows.

### Step 8 — Analytics Module
- Aggregation cron job (runs hourly) rolling up Redis live counters into `usage_daily`.
- `GET /apps/:appId/analytics?range=7d` — returns time-series for dashboard charts.
- `GET /apps/:appId/analytics/live` — Server-Sent Events or polling endpoint for real-time connection count.

### Step 9 — Webhooks Module
- `POST /apps/:appId/webhooks` — register endpoint URL + event types.
- `PATCH/DELETE /apps/:appId/webhooks/:id`.
- `WebhookDispatcherService` — subscribes to Soketi events, enqueues BullMQ jobs, signs payload with webhook secret, POSTs to tenant URL, retries with exponential backoff (max 5 attempts).
- `GET /apps/:appId/webhooks/:id/deliveries` — delivery logs for debugging.

### Step 10 — Rate Limit Module
- Redis-backed sliding window counters per `app_id`.
- `RateLimitGuard` applied to public API endpoints (message publishing via REST trigger endpoint).
- Returns `429 Too Many Requests` with `Retry-After` header when exceeded.

### Step 11 — Billing Module (readiness only)
- `GET /billing/usage-summary` — maps `usage_daily` into plan-comparable units.
- Stub `StripeService` interface (methods not yet wired to live Stripe in v1, but contracts defined for Phase 2).

### Step 12 — Pusher-Compatible Auth Endpoint
- `POST /pusher/auth` — implements Section 8.2 signing so tenants can optionally proxy through our platform instead of hosting their own auth endpoint (convenience mode).

### Step 13 — Global concerns
- Global `ValidationPipe` (class-validator DTOs on every endpoint).
- Global `HttpExceptionFilter` for consistent error JSON shape.
- `LoggingInterceptor` writing structured logs (JSON) for observability.
- Swagger/OpenAPI setup at `/api/docs`.

---

## 11. WebSocket Server (Soketi) Setup

Soketi is an open-source, high-performance, Pusher-protocol-compatible WebSocket server built on Node.js/uWebSockets.js. Rather than using its default static `apps.json` configuration (which only supports a fixed list of apps), this project implements a **custom App Provider** so Apps can be created dynamically from the dashboard without restarting the WebSocket server.

### Step 1 — Install & run Soketi
```bash
npm install -g @soketi/soketi
soketi start --config=config/soketi.config.js
```

### Step 2 — Custom App Provider (dynamic, DB-backed)
```js
// websocket/app-provider/mysql-app-provider.js
class MysqlAppProvider {
  async findById(appId) {
    // Query MySQL: apps JOIN app_keys WHERE app_id = ? AND is_active = true
    // Return shape Soketi expects: { id, key, secret, maxConnections, enableClientMessages, ... }
  }
  async findByKey(key) {
    // Same lookup by public key, used during handshake
  }
}
module.exports = MysqlAppProvider;
```

### Step 3 — soketi.config.js
```js
module.exports = {
  port: 6001,
  appManager: {
    driver: 'mysql-app-provider', // custom driver registered here
  },
  cors: { credentials: true, origin: ['*'] },
  presence: { maxMembersPerChannel: 100 },
  rateLimit: {
    enabled: true,
  },
  webhooks: {
    // Soketi's own webhook forwarding — points at our NestJS internal listener
    url: 'http://api:3000/internal/soketi-events',
    events: ['channel_occupied','channel_vacated','member_added','member_removed','client_event'],
  },
};
```

### Step 4 — Validate protocol compatibility
- Test against the official `pusher-js` client library with zero modification, only pointing `wsHost`/`wsPort`/`key` at our Soketi instance.
- Confirm public, private, and presence channel subscription flows all succeed end-to-end.

### Step 5 — Horizontal scaling
- Run multiple Soketi instances behind a load balancer with sticky sessions (WebSocket requires session affinity) OR
- Configure Soketi's Redis adapter so multiple instances share presence/channel state and can broadcast across nodes.

---

## 12. Pusher-Compatible Protocol Implementation

To guarantee drop-in compatibility, the platform must correctly implement:

| Protocol Piece | Requirement |
|---|---|
| Connection handshake | `wss://host/app/{key}?protocol=7&client=js&version=8.x` |
| Public channels | No auth required, prefix-free names (e.g. `chat`) |
| Private channels | Prefix `private-`, requires signed auth (Section 8.2) |
| Presence channels | Prefix `presence-`, requires signed auth + `channel_data` (user_id, user_info) |
| Client events | Prefix `client-`, only allowed if `enable_client_events = true` on the App |
| Trigger API (server → channel) | `POST /apps/{app_id}/events` with HMAC-signed request (App key/secret), matches Pusher's REST trigger spec |
| Batch events | `POST /apps/{app_id}/batch_events` |
| Channel info API | `GET /apps/{app_id}/channels` and `GET /apps/{app_id}/channels/{channel_name}` |

This means any existing Pusher SDK (JS, iOS, Android, Laravel, Rails) can be pointed at our platform simply by overriding the `host`/`wsHost`/`httpHost` configuration option — no code rewrite required on the tenant's side.

---

## 13. Admin Dashboard (Next.js) Development Steps

### Step 1 — Project bootstrap
```bash
npx create-next-app@latest admin-dashboard --typescript --tailwind --app
cd admin-dashboard
npm install @tanstack/react-query axios recharts zustand
npm install shadcn-ui && npx shadcn-ui init
```

### Step 2 — Auth pages
- `/login`, `/register` — forms calling NestJS `/auth/login` and `/auth/register`.
- `middleware.ts` — protects `/dashboard/**` routes, redirects unauthenticated users to `/login`.
- Store access token in a React context / Zustand store (memory only); refresh silently via httpOnly cookie.

### Step 3 — Dashboard shell/layout
- Sidebar: Apps, Analytics, Webhooks, Billing, Settings, (Super Admin section if role allows).
- Topbar: tenant name, plan badge, logout.

### Step 4 — Apps list page (`/dashboard/apps`)
- Table of Apps: name, app_id, status, created date, live connections (polled).
- "Create App" modal (Section 9.2 flow).

### Step 5 — App detail page (`/dashboard/apps/[appId]`)
- Tabs: Overview, Keys, Channels, Analytics, Webhooks, Rate Limits, Settings.
- **Overview tab:** live connection count widget, quick-start code snippet (auto-filled with real `key`/`cluster`).
- **Keys tab:** shows `app_id` and `key` (always visible), `secret` (masked, "reveal once" pattern), "Rotate Secret" button.
- **Channels tab:** list of active/known channels with type badges (public/private/presence), last active timestamp.
- **Analytics tab:** charts (Recharts) — connections over time, messages/day, peak concurrency, bandwidth.
- **Webhooks tab:** table of configured webhooks, add/edit/delete, delivery log viewer with response codes.
- **Rate Limits tab:** shows current plan-based limits, upgrade CTA if near threshold.
- **Settings tab:** rename app, toggle client events, delete app.

### Step 6 — Usage/Billing page (`/dashboard/billing`)
- Current plan card, usage-vs-limit progress bars, "Upgrade Plan" CTA (Stripe checkout stub).

### Step 7 — Multi-user management (`/dashboard/settings/team`)
- Invite teammate by email (role selection: admin/member).
- Pending invites list, revoke access, change role.

### Step 8 — Super Admin section (`/admin`, role-gated)
- List all tenants, suspend/reactivate tenant, view platform-wide usage totals, impersonate-tenant (support tooling) with full audit logging.

### Step 9 — Real-time widgets
- Use our OWN platform (dogfooding) — the dashboard subscribes to a private analytics channel via the Pusher-compatible SDK to show live connection counters without polling.

---

## 14. Feature-by-Feature Breakdown

### ✅ Admin Dashboard
Central UI for tenants and super-admins. Built in Next.js with server components for data-heavy pages and client components for interactive widgets (charts, live counters).

### ✅ Create Multiple Apps (Tenants)
Each tenant can create unlimited Apps (subject to plan caps), each fully isolated with its own credentials, channels, and analytics.

### ✅ App ID / Key / Secret
Three-part credential system matching Pusher's model: `app_id` (internal reference), `key` (public, embedded in frontend code), `secret` (private, used server-side to sign requests — never exposed to browsers).

### ✅ Usage Analytics
Per-App and per-tenant rollups: connections, messages, peak concurrency, bandwidth — visualized with day/week/month ranges.

### ✅ API Key Management
Create, view (masked), rotate (with grace period), and revoke keys without downtime.

### ✅ WebSocket Server
Soketi-powered, horizontally scalable, dynamic multi-tenant app resolution via custom MySQL-backed app provider.

### ✅ Pusher-Compatible API
Full protocol parity (Section 12) so existing Pusher SDKs work without modification.

### ✅ Multi-User Management
Tenant accounts support multiple team members with roles (owner, admin, member), invite flow, and access revocation.

### ✅ Multi-Tenant (Mandatory)
Strict isolation enforced at both the database layer (tenant_id scoping) and the WebSocket layer (per-App key/secret resolution).

### ✅ Channel Analytics
Per-channel breakdown: type, peak members (presence), message volume, last active.

### ✅ Webhooks
Event-driven notifications to tenant-owned endpoints for channel/member lifecycle events, with signed payloads and delivery retry logs.

### ✅ Rate Limits
Per-App configurable limits on messages/sec, connections/min, and API requests/min, enforced via Redis sliding-window counters.

### ✅ JWT Auth
Dashboard session security via short-lived access tokens + httpOnly refresh tokens.

### ✅ Billing Ready
Usage data structured to map directly onto Stripe metered billing or manual invoicing, without re-architecting the data model later.

---

## 15. API Endpoint Reference

### Auth
```
POST   /auth/register
POST   /auth/login
POST   /auth/refresh
POST   /auth/logout
GET    /auth/me
```

### Tenants
```
GET    /tenants/me
PATCH  /tenants/me
GET    /tenants               (super_admin only)
PATCH  /tenants/:id/status    (super_admin only, suspend/activate)
```

### Apps
```
POST   /apps
GET    /apps
GET    /apps/:appId
PATCH  /apps/:appId
DELETE /apps/:appId
POST   /apps/:appId/rotate-secret
```

### Channels
```
GET    /apps/:appId/channels
GET    /apps/:appId/channels/:channelName/stats
```

### Analytics
```
GET    /apps/:appId/analytics?range=7d
GET    /apps/:appId/analytics/live
GET    /tenants/me/analytics/summary
```

### Webhooks
```
POST   /apps/:appId/webhooks
GET    /apps/:appId/webhooks
PATCH  /apps/:appId/webhooks/:id
DELETE /apps/:appId/webhooks/:id
GET    /apps/:appId/webhooks/:id/deliveries
```

### Pusher-Compatible (used by tenant's client SDKs / server SDKs)
```
POST   /pusher/auth
POST   /apps/:appId/events
POST   /apps/:appId/batch_events
GET    /apps/:appId/channels        (Pusher-spec channel info format)
```

### Billing
```
GET    /billing/usage-summary
GET    /billing/plans
POST   /billing/checkout            (stub, Phase 2)
```

### Users / Team
```
POST   /users/invite
GET    /users
PATCH  /users/:id/role
DELETE /users/:id
```

---

## 16. Rate Limiting Strategy

Rate limiting exists at two levels:

1. **API-level rate limiting** — protects the NestJS REST API from abuse (e.g., brute-force login attempts, excessive polling). Implemented with a Redis-backed sliding window per `tenant_id` + IP.
2. **App-level real-time rate limiting** — protects the WebSocket layer from a single tenant's App overwhelming shared infrastructure. Implemented via `rate_limit_policies` per App, enforced by Soketi's built-in rate limiter plus a NestJS guard on the REST "trigger event" endpoint.

### Example policy defaults by plan

| Plan | Max Connections | Messages/sec | API req/min |
|---|---|---|---|
| Free | 100 | 10 | 60 |
| Starter | 1,000 | 50 | 300 |
| Pro | 10,000 | 200 | 1,000 |
| Enterprise | Custom | Custom | Custom |

When a limit is exceeded:
- REST API returns `429 Too Many Requests` with `Retry-After` header.
- WebSocket connections beyond `max_connections` are rejected at handshake with a Pusher-spec error code (`4004` — over connection quota, matching Pusher's own error code scheme).

---

## 17. Usage Analytics System

### Data Pipeline

```
Soketi Event (connection, message, disconnect)
        │
        ▼
Redis counters (INCR per app_id, per channel, per minute bucket)
        │
        ▼ (hourly cron)
NestJS AnalyticsAggregatorService
        │
        ▼
MySQL usage_daily / channel_stats (durable rollups)
        │
        ▼
Dashboard charts (Recharts) via GET /apps/:appId/analytics
```

### Metrics Tracked

- Concurrent connections (live + peak/day)
- Messages published per channel per day
- Bandwidth (approx bytes transferred, based on payload size)
- Channel type breakdown (public vs private vs presence)
- Top channels by activity
- API request volume (for API-level analytics, separate from WS analytics)

### Live Metrics

For "live" widgets (e.g., current connection count on the App overview page), the dashboard reads directly from Redis (via a lightweight `GET /apps/:appId/analytics/live` endpoint) rather than waiting for the hourly MySQL rollup, ensuring near real-time accuracy.

---

## 18. Webhooks System

Webhooks let tenants react server-side to real-time events without needing to run their own WebSocket listener.

### Supported Event Types

- `channel_occupied` — first subscriber joins a channel
- `channel_vacated` — last subscriber leaves a channel
- `member_added` — a member joins a presence channel
- `member_removed` — a member leaves a presence channel
- `client_event` — a client-originated event was published (only if `enable_client_events = true`)

### Delivery Flow

```
1. Soketi detects event → posts internally to NestJS /internal/soketi-events
2. NestJS looks up all active webhooks for that app_id + event type
3. For each match:
     a. Build payload: { app_id, event, channel, data, timestamp }
     b. Sign payload: X-Signature header = HMAC_SHA256(webhook.secret, payload)
     c. Enqueue BullMQ job -> WebhookDeliveryProcessor
4. Processor POSTs to tenant's registered URL
5. On non-2xx response or timeout: retry with exponential backoff (1m, 5m, 15m, 1h, 6h) up to 5 attempts
6. Every attempt logged in webhook_deliveries for the tenant to inspect in the dashboard
```

### Tenant-Side Verification

Tenants are instructed to verify `X-Signature` against their webhook secret before trusting payloads — identical to how Stripe/Pusher webhook verification works, giving tenants a familiar integration pattern.

---

## 19. Billing Readiness

While full billing is out of scope for v1, the schema and services are deliberately structured so billing can be turned on without a data migration:

- `usage_daily` is already the exact shape needed to report metered usage to Stripe (`stripe.subscriptionItems.createUsageRecord`).
- `billing_accounts` table stores `stripe_customer_id` ahead of time.
- Plans (`free`, `starter`, `pro`, `enterprise`) already gate `rate_limit_policies` and `max_connections`, so upgrading a tenant's plan immediately changes their enforced limits — the same mechanism billing would use to unlock capacity after a successful payment.
- `GET /billing/usage-summary` returns tenant usage in a Stripe-metered-billing-friendly shape: `{ period_start, period_end, connections_used, messages_used, overage_units }`.

**Phase 2 (future work, not in this documentation's build scope):** wire `POST /billing/checkout` to Stripe Checkout, add Stripe webhook listener (`invoice.paid`, `customer.subscription.updated`) to sync `tenants.plan` automatically.

---

## 20. Security Considerations

- **Secrets encrypted at rest** — `app_keys.secret` and `webhooks.secret` stored using AES-256-GCM, decrypted only in-memory when needed for signing/verification.
- **Password hashing** — bcrypt with cost factor 12 for `users.password_hash`.
- **JWT best practices** — short-lived access tokens (15 min), httpOnly + Secure + SameSite=Strict refresh cookie, refresh token rotation on every use, blacklist on logout.
- **Tenant isolation guard** — every DB query passes through `TenantScopeGuard`; unit tests specifically assert that cross-tenant ID access returns 404, not data leakage.
- **Rate limiting on auth endpoints** — prevents brute-force login attempts.
- **HTTPS/WSS only in production** — enforced at the load balancer/ingress level.
- **Input validation** — every DTO uses `class-validator` decorators; no raw request bodies reach services.
- **Audit logging** — super-admin actions (impersonation, tenant suspension) are logged with actor, timestamp, and target for compliance.
- **CORS** — Soketi and NestJS both restrict allowed origins per environment; wildcard `*` only used in local dev.
- **Dependency scanning** — CI pipeline runs `npm audit` / Dependabot on every PR.

---

## 21. Development Environment Setup

### Prerequisites
- Node.js 20+
- MySQL 8+ (installed natively or via a managed instance)
- Redis 7+ (installed natively or via a managed instance)
- PM2 (`npm install -g pm2`) for running/supervising multiple Node processes locally and in production

### Step-by-step local setup

```bash
# 1. Clone monorepo
git clone <repo-url> realtime-saas
cd realtime-saas

# 2. Install dependencies (root, uses workspaces)
npm install

# 3. Start supporting services (installed natively on the machine)
sudo service mysql start
sudo service redis-server start
# On macOS with Homebrew: brew services start mysql && brew services start redis

# 4. Configure environment variables
cp apps/api/.env.example apps/api/.env
cp apps/admin-dashboard/.env.example apps/admin-dashboard/.env.local

# 5. Run database migrations
cd apps/api
npx prisma migrate dev
npx prisma db seed   # optional: seeds a demo tenant + app

# 6. Start all three services in parallel (from repo root)
npm run dev
#   -> admin-dashboard on http://localhost:3001
#   -> api             on http://localhost:3000
#   -> websocket       on ws://localhost:6001
```

### Sample `.env` (api)
```
DATABASE_URL="mysql://root:password@localhost:3306/realtime_saas"
REDIS_URL="redis://localhost:6379"
JWT_ACCESS_SECRET="change-me"
JWT_REFRESH_SECRET="change-me-too"
ENCRYPTION_KEY="32-byte-hex-key-for-aes-256-gcm"
SOKETI_INTERNAL_URL="http://localhost:6001"
```

### Sample `.env.local` (admin-dashboard)
```
NEXT_PUBLIC_API_URL="http://localhost:3000"
NEXT_PUBLIC_WS_HOST="localhost"
NEXT_PUBLIC_WS_PORT="6001"
```

---

## 22. Testing Strategy

| Test Type | Tooling | Coverage Target |
|---|---|---|
| Unit tests (services/guards) | Jest | 80%+ on `auth`, `apps`, `rate-limit` modules |
| Integration tests (API + DB) | Jest + Testcontainers (MySQL) | All critical endpoints (auth, app CRUD, secret rotation) |
| WebSocket protocol tests | `pusher-js` client against local Soketi | Public/private/presence subscribe flows |
| E2E tests (dashboard) | Playwright | Login → create app → view keys → rotate secret → delete app |
| Load tests | k6 or Artillery | Simulate 10k concurrent WS connections on a single App |
| Security tests | OWASP ZAP baseline scan | Run against staging before each release |

### Critical Test Scenarios

1. Tenant A cannot fetch Tenant B's App details even with a guessed valid `appId` (expect 404).
2. Rotating a secret does not immediately break already-connected clients (grace period honored).
3. Deleting an App force-disconnects all live sockets for that `app_id` within a defined SLA (e.g., <5s).
4. Webhook delivery retries follow the defined backoff schedule and stop after 5 attempts.
5. Rate limit returns `429` exactly at the configured threshold, not before or after.
6. Presence channel `member_added`/`member_removed` webhook events fire with correct `user_info` payload.

---

## 23. Deployment Steps

No containers are used anywhere in this project. All three services run as **plain Node.js processes**, supervised by **PM2** (or `systemd` as an alternative), and exposed to the internet through an **Nginx reverse proxy**. MySQL and Redis run as natively installed services (or managed cloud instances) rather than containers.

### Step 1 — Build each service on the target server
```bash
# On the deployment server (or CI runner producing a build artifact)
git clone <repo-url> realtime-saas
cd realtime-saas
npm install

# Build each app
npm run build --workspace=apps/api
npm run build --workspace=apps/admin-dashboard
# websocket (Soketi) requires no build step, only its config/app-provider files
```

### Step 2 — Environment configuration
- Copy `.env` files onto the server (Section 21) with production values — real MySQL/Redis hosts, strong JWT secrets, production encryption key.
- Never commit `.env` files to version control; manage them via the server's secret store or a tool like `direnv`/`dotenv-vault`.

### Step 3 — Process management with PM2

`ecosystem.config.js` (repo root):
```js
module.exports = {
  apps: [
    {
      name: 'api',
      cwd: './apps/api',
      script: 'dist/main.js',
      instances: 2,              // cluster mode, one per CPU core as needed
      exec_mode: 'cluster',
      env_production: { NODE_ENV: 'production', PORT: 3000 },
    },
    {
      name: 'websocket',
      cwd: './apps/websocket',
      script: 'node_modules/.bin/soketi',
      args: 'start --config=config/soketi.config.js',
      instances: 1,               // scale by running additional instances on separate ports/servers behind the LB
      env_production: { NODE_ENV: 'production' },
    },
    {
      name: 'admin-dashboard',
      cwd: './apps/admin-dashboard',
      script: 'node_modules/.bin/next',
      args: 'start -p 3001',
      instances: 2,
      exec_mode: 'cluster',
      env_production: { NODE_ENV: 'production' },
    },
  ],
};
```

Start and manage everything with:
```bash
pm2 start ecosystem.config.js --env production
pm2 save                # persist process list across reboots
pm2 startup              # generates the systemd/init script to auto-launch PM2 on server boot
pm2 status                # view running processes
pm2 logs api               # tail logs for a specific service
pm2 reload api              # zero-downtime reload after a new deploy
```

### Step 4 — Nginx reverse proxy & TLS termination
```nginx
# /etc/nginx/sites-available/realtime-saas

server {
  listen 443 ssl;
  server_name dashboard.ourplatform.com;
  ssl_certificate     /etc/letsencrypt/live/ourplatform.com/fullchain.pem;
  ssl_certificate_key  /etc/letsencrypt/live/ourplatform.com/privkey.pem;

  location / {
    proxy_pass http://127.0.0.1:3001;
    proxy_set_header Host $host;
  }
}

server {
  listen 443 ssl;
  server_name api.ourplatform.com;
  ssl_certificate     /etc/letsencrypt/live/ourplatform.com/fullchain.pem;
  ssl_certificate_key  /etc/letsencrypt/live/ourplatform.com/privkey.pem;

  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_set_header Host $host;
  }
}

server {
  listen 443 ssl;
  server_name ws.ourplatform.com;
  ssl_certificate     /etc/letsencrypt/live/ourplatform.com/fullchain.pem;
  ssl_certificate_key  /etc/letsencrypt/live/ourplatform.com/privkey.pem;

  location / {
    proxy_pass http://127.0.0.1:6001;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;    # required for WebSocket upgrade
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 86400s;                  # keep long-lived WS connections alive
  }
}
```
TLS certificates are obtained/renewed via Certbot (`certbot --nginx`).

### Step 5 — Horizontal scaling without containers
- **API:** run PM2 in `cluster` mode (as shown above) to use all CPU cores on a single server; for multi-server scaling, run the same `pm2 start` setup on additional servers and place them behind a load balancer (Nginx upstream or a managed LB), pointed at the same MySQL/Redis.
- **WebSocket (Soketi):** run one Soketi process per server/port; place multiple Soketi instances behind a load balancer configured for **sticky sessions** (WebSocket requires session affinity), and enable Soketi's Redis adapter so presence/channel state is shared across instances.
- **Admin Dashboard:** stateless Next.js processes scale the same way as the API — multiple PM2 cluster instances behind Nginx/a load balancer.
- **MySQL:** run on its own server (or managed database service) with a read replica for analytics-heavy queries as usage grows.
- **Redis:** run on its own server (or managed Redis service) for rate-limit counters and pub/sub.

### Step 6 — CI/CD Pipeline (no image registry needed)
```
1. PR opened → lint + unit tests + build (GitHub Actions runner)
2. Merge to main → build artifacts (npm run build) → package as a tarball/rsync bundle
3. Deploy script copies the build to the target server(s) via rsync/scp over SSH
4. Remote script runs: npm install --omit=dev, then `pm2 reload ecosystem.config.js --env production`
   (zero-downtime reload; PM2 restarts workers one at a time)
5. Post-deploy smoke test: curl health checks on /health for api and the websocket server
6. Manual approval gate before the production deploy step for main branch merges
```

### Step 7 — Observability
- `/health` and `/ready` endpoints on both `api` and `websocket` (checked directly via `pm2` health hooks or an external uptime monitor).
- Structured JSON logs written to disk and rotated with `pm2-logrotate`, optionally shipped to a log aggregator (e.g., a hosted Loki/ELK instance) via a lightweight log-forwarding agent.
- Prometheus `node_exporter`/custom metrics endpoint: active connections, message throughput, webhook delivery success rate, API latency (p50/p95/p99).
- Alerting on: connection count near plan-wide capacity, webhook delivery failure rate > 5%, API error rate > 1%, or any PM2 process entering a restart loop.

---

## 24. Expected Outputs Per Phase

### Phase 1 — Foundations (Weeks 1–2)
**Expected Output:** Monorepo scaffolded; MySQL schema migrated; NestJS auth module working (register/login/refresh); Next.js login/register pages functional against real API; MySQL and Redis running as native local services (or managed cloud instances).

### Phase 2 — Core Tenant/App Management (Weeks 3–4)
**Expected Output:** Tenants can create/list/view/delete Apps from the dashboard; App ID/Key/Secret generated and displayed correctly (secret shown once); secret rotation working with grace period; all endpoints tenant-scoped and verified via tests.

### Phase 3 — WebSocket Integration (Weeks 5–6)
**Expected Output:** Soketi running with custom MySQL app-provider; a test HTML page using `pusher-js` can successfully connect, subscribe to public/private/presence channels against a dynamically created App; `/pusher/auth` endpoint correctly signs channel auth requests.

### Phase 4 — Analytics & Channels (Weeks 7–8)
**Expected Output:** Channel list + stats visible per App; live connection counter on dashboard updating in near real-time; hourly aggregation job populating `usage_daily`; analytics charts rendering real historical data.

### Phase 5 — Webhooks & Rate Limiting (Weeks 9–10)
**Expected Output:** Tenants can register webhook URLs; test events delivered with correct signature and retried on failure; rate limits enforced at both API and WebSocket layers with correct `429`/connection-rejection behavior.

### Phase 6 — Multi-User & Super Admin (Week 11)
**Expected Output:** Team invite flow functional; role-based access enforced; super-admin panel lists all tenants and can suspend/reactivate accounts with audit log entries.

### Phase 7 — Billing Readiness & Hardening (Week 12)
**Expected Output:** `usage-summary` endpoint returns billing-shaped data; security checklist (Section 20) fully verified; load test confirms target concurrent connection count per plan tier; staging deployment stable for 72 hours before production cutover.

### Final Expected Deliverable
A fully functioning, containerized, multi-tenant real-time SaaS platform where:
- Any number of customers can self-serve create Apps.
- Each App is fully isolated and Pusher-protocol compatible.
- Usage, channels, and connections are observable in a polished Next.js dashboard.
- Webhooks and rate limits behave predictably and are documented for tenant developers.
- The system is ready for a billing integration to be layered on top without schema rework.

---

## 25. Project Roadmap / Milestones

| Milestone | Target Outcome |
|---|---|
| M1 | Auth + Tenant + App CRUD complete |
| M2 | Soketi dynamic app-provider integrated and protocol-verified |
| M3 | Dashboard fully functional for App/Key management |
| M4 | Analytics pipeline live with real charts |
| M5 | Webhooks + Rate limiting shipped |
| M6 | Multi-user + Super Admin shipped |
| M7 | Security hardening + load testing passed |
| M8 | Production launch (v1.0) |
| M9 (Phase 2, post-launch) | Stripe billing fully wired |
| M10 (Phase 2, post-launch) | Multi-region WebSocket clustering |

---

## 26. Appendix: Sample Configs & Requests

### Sample: Create App Request
```http
POST /apps
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "name": "Customer A - Production",
  "cluster": "mt1",
  "enable_client_events": false
}
```

### Sample: Create App Response
```json
{
  "app_id": "app_9f8a2b31cd",
  "key": "8a72e1c9f4d6b0a3",
  "secret": "s3cr3t_shown_only_once_store_it_safely",
  "cluster": "mt1",
  "status": "active",
  "created_at": "2026-07-18T10:15:00Z"
}
```

### Sample: Pusher-Compatible Client Connection (Frontend, Tenant's Product)
```js
import Pusher from 'pusher-js';

const pusher = new Pusher('8a72e1c9f4d6b0a3', {
  wsHost: 'ws.ourplatform.com',
  wsPort: 443,
  forceTLS: true,
  cluster: 'mt1',
  authEndpoint: 'https://tenant-backend.example.com/pusher/auth',
});

const channel = pusher.subscribe('private-orders-55');
channel.bind('order-updated', (data) => {
  console.log('Order updated:', data);
});
```

### Sample: Trigger Event (Server-Side, Tenant's Backend)
```http
POST /apps/app_9f8a2b31cd/events
X-Signature: <hmac-sha256-signature>
Content-Type: application/json

{
  "channel": "orders",
  "event": "order-updated",
  "data": { "order_id": 55, "status": "shipped" }
}
```

### Sample: Webhook Payload Sent to Tenant
```json
{
  "app_id": "app_9f8a2b31cd",
  "event": "member_added",
  "channel": "presence-support",
  "data": { "user_id": "u_2291", "user_info": { "name": "Asha" } },
  "timestamp": "2026-07-18T10:20:31Z"
}
```

### Sample: Rate Limit Response
```http
HTTP/1.1 429 Too Many Requests
Retry-After: 12

{
  "statusCode": 429,
  "message": "Rate limit exceeded for app_9f8a2b31cd: max 10 messages/sec on Free plan",
  "error": "Too Many Requests"
}
```

---

**End of Document**

This documentation is intended as a living reference. As Phase 2 features (Stripe billing, multi-region clustering, native mobile SDKs) are implemented, this document should be versioned and updated alongside the codebase.
