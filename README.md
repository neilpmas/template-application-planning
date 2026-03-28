# Template Application

A base template used to spin up new projects with a consistent, repeatable tech stack.

## Architecture Overview

```
┌─────────────────────────────────────────────────┐
│                  Cloudflare                      │
│  ┌─────────────┐      ┌──────────────────────┐  │
│  │  React App  │ ───► │  BFF (CF Workers)    │  │
│  └─────────────┘      └──────────┬───────────┘  │
└─────────────────────────────────────────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────┐
                    │  Spring Boot on Fly.io   │
                    └──────────────────────────┘
                                   │
                         ┌─────────┴─────────┐
                         │                   │
                         ▼                   ▼
                     Supabase             Auth0
                   (Database)        (Authentication)
```

## Stack

### Frontend
- **React** — web UI
- **React Native** — planned for mobile apps
- **Hosted on**: Cloudflare Pages

### Backend for Frontend (BFF)
- **Cloudflare Workers** — lightweight proxy layer between frontend and backend
- Handles request routing, auth token forwarding, and response shaping

### Backend
- **Spring Boot** (Java)
- **Hosted on**: Fly.io

### Database
- **Supabase** — Postgres-based, managed database

### Authentication
- **Auth0** — identity and access management

## Authentication Detail

### Approach: BFF-based OAuth (OAuth 2.0 for Browser-Based Apps, BCP212)

This template follows the BFF pattern as recommended by the OAuth 2.0 for Browser-Based Apps spec. The key principle is that **JWTs never touch the browser** — the BFF owns the OAuth flow and issues a session cookie to the frontend instead.

This eliminates token exposure in the browser (no localStorage, no JS-accessible tokens).

### Auth0 Tenant Setup

Each project gets its own Auth0 tenant (or application within a shared tenant). Two Auth0 applications are registered:

- **Regular Web Application** — for the BFF/Cloudflare Worker (Authorization Code + PKCE, has a client secret)
- **API** — represents the Spring Boot backend, defines the audience

Key Auth0 config values needed per project:

| Setting | Description |
|---|---|
| `Domain` | e.g. `your-tenant.auth0.com` |
| `Client ID` | Web application client ID |
| `Client Secret` | Web application client secret (held only by the BFF) |
| `Audience` | API identifier, e.g. `https://api.yourproject.com` |

### Frontend (React)

The frontend holds **no tokens**. It:

- Redirects to the BFF's `/auth/login` endpoint to initiate login
- Receives an `HttpOnly; Secure; SameSite=Strict` session cookie from the BFF on completion
- Makes all API calls to the BFF using the session cookie (no Authorization header needed)
- Calls the BFF's `/auth/logout` endpoint to end the session

### BFF (Cloudflare Workers)

The BFF is the OAuth client — it owns the full auth flow. Implemented using [`oauth4webapi`](https://github.com/panva/oauth4webapi), which is spec-compliant and runs natively in the Workers runtime (uses Web Crypto API, no Node.js dependencies).

**Login flow:**
1. React redirects to BFF `/auth/login`
2. BFF redirects user to Auth0 with Authorization Code + PKCE
3. Auth0 redirects back to BFF `/auth/callback` with the code
4. BFF exchanges the code for access + refresh tokens (using client secret)
5. BFF stores tokens in **Cloudflare KV** (keyed by a generated session ID)
6. BFF issues an `HttpOnly` session cookie to the browser and redirects to the app

**Per-request flow:**
1. React sends request to BFF with session cookie
2. BFF looks up the session in KV, retrieves the access token
3. BFF validates the JWT (using Auth0 JWKS via Web Crypto API)
4. If the token is expired, BFF uses the refresh token to obtain a new one and updates KV
5. BFF forwards the request to Spring Boot with `Authorization: Bearer <token>`

**Session storage (Cloudflare KV):**
- Session ID → `{ accessToken, refreshToken, expiresAt }`
- KV TTL aligned with refresh token lifetime
- KV is eventually consistent — acceptable for session reads

### Backend (Spring Boot)

Spring Boot validates the JWT on every protected request. It trusts only requests from the BFF (internal network on Fly.io):

- Uses `spring-boot-starter-oauth2-resource-server`
- Configured with the Auth0 `issuer-uri` and `audience`
- JWT is validated against Auth0's JWKS endpoint automatically

`application.yml` config:

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://your-tenant.auth0.com/
          audiences: https://api.yourproject.com
```

Protect endpoints using standard Spring Security annotations e.g. `@PreAuthorize("isAuthenticated()")`.

### Roles & Permissions

- Roles and permissions are defined in Auth0 and included in the JWT as a custom claim (e.g. `permissions`)
- Enable **RBAC** and **Add Permissions in the Access Token** in the Auth0 API settings
- Spring Boot reads the `permissions` claim to enforce fine-grained access control

### Auth Flow Summary

```
User → React → BFF /auth/login → Auth0 (Authorization Code + PKCE)
                                        │
                                   code returned
                                        │
                    BFF exchanges code → tokens stored in KV
                    BFF issues HttpOnly session cookie → React
                                        │
React (cookie) → BFF → validates JWT → Spring Boot (Bearer token)
```

## BFF Auth Library

The BFF auth layer is provided by **[Portcullis](../portcullis/README.md)** — a standalone open source Cloudflare Workers auth library built for this stack.

Until Portcullis is built, the BFF auth layer is generated per project.

## Principles

- Same stack across every project for consistency and reuse
- BFF pattern keeps the frontend decoupled from backend changes
- Git is the source of truth for versioning — no manual version labels
