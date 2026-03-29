# Template Application — Planning

Architecture, decisions, and workflow guide for the template application stack. This repo is the starting point for every new project built on this stack.

## Repos

| Repo | Purpose |
|---|---|
| [template-application-planning](https://github.com/neilpmas/template-application-planning) | This repo — architecture, decisions, workflow guide |
| [template-application-frontend](https://github.com/neilpmas/template-application-frontend) | React + Cloudflare Workers BFF |
| [template-application-backend](https://github.com/neilpmas/template-application-backend) | Spring Boot backend |

---

## Architecture Overview

```
                    ┌─────────────────────────────────────────────────┐
                    │                  Cloudflare                      │
                    │  ┌─────────────┐      ┌──────────────────────┐  │
  Browser ────────► │  │  React App  │ ───► │  BFF (CF Workers)    │  │
  (HTTP/3)          │  └─────────────┘      └──────────┬───────────┘  │
                    └─────────────────────────────────────────────────┘
                                                        │ gRPC-Web (HTTP/2)
                                                        ▼
  Mobile/Desktop ──────────────────────────────────────►
  (Bearer token, direct)                  ┌─────────────────────────┐
                                          │  Spring Boot on Fly.io  │
                                          │  (Spring Modulith)      │
                                          └─────────────────────────┘
                                                        │
                                             ┌──────────┴──────────┐
                                             │                     │
                                             ▼                     ▼
                                         Supabase               Auth0
                                       (Database)          (Authentication)
```

### Client model

The BFF exists solely to protect browser-based clients, where tokens cannot be stored safely. Mobile and desktop clients have secure OS credential stores and talk directly to Spring Boot.

| Client | Auth pattern | Talks to |
|---|---|---|
| React (web) | Session cookie via BFF (JWTs never touch browser) | BFF → Spring Boot |
| React Native (mobile) | Bearer token direct from Auth0 | Spring Boot directly |
| Desktop | Bearer token direct from Auth0 | Spring Boot directly |

Spring Boot validates JWTs from all clients the same way — it doesn't distinguish between BFF-forwarded and direct requests.

---

## Stack

### Frontend & BFF — Turborepo monorepo
- **Turborepo** — monorepo managing web, mobile, and BFF as packages
- **React** — web UI, hosted on Cloudflare Pages
- **Vite** — build tool
- **Tailwind CSS** — utility-first styling
- **shadcn/ui** — component library (copy-paste, Radix UI primitives, you own the code)
- **React Native** — iOS and Android (Auth0 RN SDK, `expo-secure-store`)
- **Cloudflare Workers** — BFF, auth proxy, request routing (web only)
- **Hono** — router for the BFF
- **[Bezzie](https://github.com/neilpmas/bezzie)** — BFF OAuth 2.0 library (open source, built for this stack)
- **Cloudflare KV** — session storage

```
<app>/
  apps/
    web/        ← React (Cloudflare Pages)
    mobile/     ← React Native (iOS + Android)
    bff/        ← Cloudflare Workers + Bezzie
  packages/
    types/      ← shared TypeScript types
    api-client/ ← shared API client
```

### Backend
- **Spring Boot** (Java) — core business logic
- **Spring Modulith** — enforces clean module boundaries
- **Maven** — build tool
- **Hosted on**: Fly.io

### Database
- **Supabase** — Postgres-based, managed database

### Authentication
- **Auth0** — identity and access management

---

## Protocol Decisions

### Browser → Cloudflare edge
HTTP/3 — handled automatically by Cloudflare. No configuration needed.

### BFF → Spring Boot
**gRPC-Web over HTTP/2** — Cloudflare Workers can call gRPC-Web endpoints via `fetch()`. Spring Boot exposes a gRPC-Web endpoint. This gives strongly typed contracts (Protobuf), HTTP/2 efficiency, and avoids the Workers runtime limitations of native gRPC (which requires HTTP/2 trailers not accessible via `fetch()`).

> Note: [Connect protocol](https://connectrpc.com) would be the ideal long-term choice here but has no official Java implementation yet. Worth revisiting when it lands.

### Mobile/Desktop → Spring Boot
Standard HTTPS with Bearer token. Protocol TBD per client — REST is the default.

### Spring Boot → Supabase
JDBC over TCP — protocol not a concern here.

---

## Authentication Detail

### Approach: BFF-based OAuth (OAuth 2.0 for Browser-Based Apps, BCP212)

The BFF pattern keeps JWTs out of the browser entirely. The BFF owns the OAuth flow and issues a session cookie to React instead.

### Auth0 Setup

Two Auth0 applications per project:

- **Regular Web Application** — for the BFF (Authorization Code + PKCE, has a client secret)
- **API** — represents the Spring Boot backend, defines the audience

Key config values:

| Setting | Description |
|---|---|
| `Domain` | e.g. `your-tenant.auth0.com` |
| `Client ID` | Web application client ID |
| `Client Secret` | Web application client secret (held only by the BFF) |
| `Audience` | API identifier, e.g. `https://api.yourproject.com` |

### Frontend (React)

The frontend holds no tokens. It:

- Redirects to BFF `/auth/login` to initiate login
- Receives an `HttpOnly; Secure; SameSite=Strict` session cookie from the BFF on completion
- Makes all API calls to the BFF using the session cookie
- Calls BFF `/auth/logout` to end the session

### BFF (Cloudflare Workers + Bezzie)

The BFF owns the full OAuth flow.

**Login flow:**
1. React redirects to BFF `/auth/login`
2. BFF redirects to Auth0 (Authorization Code + PKCE)
3. Auth0 redirects back to BFF `/auth/callback`
4. BFF exchanges code for tokens, stores them in Cloudflare KV
5. BFF issues `HttpOnly` session cookie to the browser

**Per-request flow:**
1. React sends request to BFF with session cookie
2. BFF validates the session, refreshes the token if expired
3. BFF forwards the request to Spring Boot via gRPC-Web with Bearer token

### Backend (Spring Boot)

Spring Boot is an OAuth 2.0 resource server. It validates JWTs on every protected request — regardless of whether the request came from the BFF or a mobile/desktop client.

- Uses `spring-boot-starter-oauth2-resource-server`
- Validates against Auth0's JWKS endpoint automatically

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://your-tenant.auth0.com/
          audiences: https://api.yourproject.com
```

### Roles & Permissions

Defined in Auth0, included in the JWT as a `permissions` claim. Enable **RBAC** and **Add Permissions in the Access Token** in Auth0 API settings.

```java
@PreAuthorize("hasAuthority('read:data')")
```

### Auth Flow Summary

```
User → React → BFF /auth/login → Auth0 (Authorization Code + PKCE)
                                        │
                                   code returned
                                        │
                    BFF exchanges code → tokens stored in KV
                    BFF issues HttpOnly session cookie → React
                                        │
React (cookie) → BFF → validates session → Spring Boot (gRPC-Web + Bearer token)
```

---

## Project Workflow

This template defines the end-to-end process for starting a new project. Follow these phases in order.

### Phase 1 — Define the problem
- What does this product do?
- Who are the users?
- What is the core use case?

### Phase 2 — Domain model
- Key entities and relationships
- Data model (tables, fields)

### Phase 3 — API contract
- gRPC service definitions (.proto files)
- Endpoint list, request/response shapes
- Agreed before frontend or backend work starts

### Phase 4 — Figma
- Wireframes for key screens
- Component inventory
- Note: free Figma tier (3 pages per file)

### Phase 5 — Backend scaffold
- Spring Boot + Maven + Spring Modulith
- Module structure defined upfront (e.g. `auth`, `api`, `domain`, `infrastructure`)
- Supabase/Postgres connection
- Auth0 resource server config
- gRPC-Web endpoint

### Phase 6 — BFF scaffold
- Cloudflare Worker + Hono + Bezzie
- Wire in gRPC-Web client for Spring Boot
- Auth flow end-to-end

### Phase 7 — Frontend scaffold
- React + Vite
- Component library decision (e.g. shadcn/ui)
- Auth flow (login, session cookie, logout)
- First protected page

### Phase 8 — Wire together
- End-to-end auth flow working
- First real API call from React → BFF → Spring Boot → Supabase

### Phase 9 — Deploy
- Backend: Fly.io
- Frontend + BFF: Cloudflare Pages + Workers
- CI/CD: GitHub Actions

---

## Testing Strategy

| Layer | Approach |
|---|---|
| Spring Boot — unit | JUnit 5, plain unit tests for domain logic (TDD) |
| Spring Boot — integration | Testcontainers — real Postgres, no mocks |
| BFF (Workers) | Vitest + `@cloudflare/vitest-pool-workers` |
| React | Vitest + React Testing Library |
| E2E | Playwright |

Notes:
- Docker must be running locally for Testcontainers
- Spring Boot 3+ has built-in Testcontainers support — minimal boilerplate
- TDD for Spring domain logic; test-after acceptable elsewhere while scaffolding

## Branching Strategy

**GitHub Flow** — `main` is always deployable, all work happens on short-lived branches.

- Branch naming: `claude/<short-description>` e.g. `claude/add-login-page`, `claude/phase-12-oauth-upgrade`
- PRs are small and focused — one thing at a time
- PRs are short-lived — merge, close, or rebase within a few days. Stale PRs are closed and reopened fresh.
- Branch protection on `main` — CI must pass before merge
- Claude works in a worktree → raises a PR → you review and merge

## CI/CD

**GitHub Actions** — all repos follow the same pattern.

### On pull request
- Lint
- Unit tests
- Integration tests (Testcontainers — Docker available on Actions runners, no extra setup)
- Build

### On merge to main
- Everything above, plus deploy

### Deploy targets
| Layer | Tool | Target |
|---|---|---|
| Backend | `flyctl` GitHub Action | Fly.io |
| Frontend + BFF | Wrangler GitHub Action | Cloudflare Pages + Workers |

### Environments
- Production only for now — merge to main deploys straight to prod
- Staging can be added per project if needed

### Database migrations
- Run automatically as part of deploy (Flyway)
- If a migration fails, deploy fails

### Secrets
- GitHub Actions secrets per repo
- Revisit centralised secrets management (e.g. Doppler) if managing many projects becomes painful

### Turborepo remote caching
- Defer for now — add if CI build times become a problem

## Local Development

### Stack
```
React (Vite dev server :5173)
    ↓
Cloudflare Workers (wrangler dev :8787)
    ↓
Spring Boot (:8080)
    ↓
Postgres (Docker :5432)
```

### How to run

**Infrastructure (Docker Compose):**
```bash
docker-compose up
```
Starts Postgres locally. Everything else runs natively for hot reload.

**Backend:**
```bash
./mvnw spring-boot:run
```

**BFF:**
```bash
wrangler dev
```
Cloudflare KV is simulated in memory by Wrangler — no real Cloudflare account needed locally.

**Frontend:**
```bash
npm run dev
```

### Auth0
No local equivalent — use a real Auth0 dev tenant (free tier). Register a separate Auth0 application for local dev so local and production credentials are isolated.

### Environment variables
Each layer has a local config file (gitignored):
- Spring Boot: `application-local.yml`
- BFF: `.dev.vars` (Wrangler convention)
- React: `.env.local` (Vite convention)

## Cost Philosophy

Apps are built to be cheap at rest. The model is: low cost until something takes off, then invest in that one.

| Service | Cost at rest | Notes |
|---|---|---|
| Fly.io | ~$1.94/month per app | Smallest machine. Java can't scale to zero — this is the floor. |
| Supabase | Free | Pauses after 1 week inactivity on free plan. Fine for early stage. |
| Auth0 | Free | 7,500 active users across all apps on free tier. |
| Cloudflare | Free | Pages + Workers free tier is generous. |

4 apps running ≈ $8/month total. If an app gets no traction after a few months, shut it down. When one takes off, scale that one.

**Rules:**
- Don't over-engineer infrastructure on day one
- Review and cull dead apps every few months
- Upgrade infrastructure for the app that's working, not all of them

## Observability

### Philosophy
Start lean. Add proper observability when an app gets real traffic.

### From day one (free, zero config)
- **Fly.io built-in logs** — backend logs, always on
- **Cloudflare Workers logs** — BFF logs, always on
- **Sentry** (free tier) — frontend error tracking, one Sentry org, one project per app

### When an app gets traction
- **Axiom** — unified log aggregation for backend + BFF + Auth0 events
- All logs tagged with `app`, `env`, `layer` — one dataset, query across everything
- **Micrometer → Grafana Cloud** — JVM metrics, request rates, error rates
- **Auth0 log streaming → Axiom** — login events, token issues, suspicious activity

### Log format (when Axiom is added)
```json
{
  "app": "my-app-name",
  "env": "production",
  "layer": "backend",
  "level": "error",
  "message": "..."
}
```

## Multi-App Strategy

When running multiple apps on the same stack:

- **One Auth0 tenant** — shared across all apps. Same user identity, SSO possible. Users don't need to re-register per app.
- **One Spring Boot + Supabase per app** — complete data isolation. Apps are independent and unrelated.
- **One Cloudflare Workers BFF per app** — each app has its own deployment.

Auth0 setup per app:
- One **Regular Web Application** (for the BFF)
- One **API** (for the Spring Boot backend, defines the audience)
- RBAC roles and permissions are scoped per API — no bleed between apps

## Principles

- Same stack across every project for consistency and reuse
- BFF pattern keeps the frontend decoupled from backend changes
- Spring Modulith enforces module boundaries from day one
- Git is the source of truth — no manual version labels
- Restart Claude between major phases to keep context clean
