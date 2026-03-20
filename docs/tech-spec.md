# Technical Design Document
## Push Notification Broadcasting Service

**Version:** 1.0  
**Status:** Draft  
**Last Updated:** 2026-03-04

---

## 1. System Overview

The Push Notification Broadcasting Service is a Go HTTP server that enables the marketing team to send FCM push notifications to segmented groups of trading platform clients. It exposes two distinct interface surfaces:

- A **machine-to-machine API** for the trading app to register device FCM tokens (OAuth2 Client Credentials, scope-validated JWT).
- A **browser UI** for the marketing team to manage topics, client assignments, and campaigns (Cloudflare Zero Trust + Okta, role-enforced server-side).

The service is deployed to two fully isolated environments — **staging** and **production** — each with its own Firebase project and Supabase PostgreSQL database.

---

## 2. Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Trading App (client)                 │
│         iOS / Android / Web — any device                │
└─────────────────┬───────────────────────────────────────┘
                  │ POST /api/v1/devices/register
                  │ Authorization: Bearer <M2M JWT>
                  ▼
┌─────────────────────────────────────────────────────────┐
│              Go HTTP Server (net/http)                  │
│                                                         │
│  ┌──────────────────┐   ┌──────────────────────────┐    │
│  │  M2M Auth        │   │  CF Access JWT Auth      │    │
│  │  Middleware      │   │  + Role Middleware       │    │
│  │  (scope=register)│   │  (viewer | sender)       │    │
│  └────────┬─────────┘   └────────────┬─────────────┘    │
│           │                          │                  │
│  ┌────────▼──────────────────────────▼─────────────┐    │
│  │              HTTP Handlers                      │    │
│  │  device · topic · client · campaign             │    │
│  └────────┬──────────────────────────┬─────────────┘    │
│           │                          │                  │
│  ┌────────▼────────┐      ┌──────────▼─────────────┐    │
│  │  Repository     │      │  FCM Client            │    │
│  │  (pgx / SQL)    │      │  (Firebase Admin SDK)  │    │
│  └────────┬────────┘      └──────────┬─────────────┘    │
└───────────┼──────────────────────────┼──────────────────┘
            │                          │
    ┌───────▼────────┐         ┌───────▼──────────┐
    │ Supabase       │         │ Firebase Cloud   │
    │ PostgreSQL     │         │ Messaging        │
    └────────────────┘         └──────────────────┘

── Browser ──────────────────────────────────────────────
    Marketing Team
         │
    Cloudflare Zero Trust (Okta IdP)
         │ CF-Access-JWT-Assertion header injected
         ▼
    GET|POST /  /clients  /topics  /campaigns
```

---

## 3. Technology Stack

| Concern | Choice | Notes |
|---|---|---|
| Language | Go 1.26+ | |
| HTTP Server | `net/http` | No third-party router |
| Templates | `html/template` + HTMX | Server-rendered partials |
| CSS | Pico CSS (CDN) | Classless, zero build step |
| Database driver | `pgx/v5` | Direct Supabase/Neon Postgres connection |
| Migrations | `golang-migrate` | SQL migration files |
| FCM | `firebase.google.com/go/v4` | Firebase Admin SDK |
| JWT validation | `github.com/MicahParks/keyfunc/v3` | JWKS caching |
| JWT parsing | `github.com/golang-jwt/jwt/v5` | |
| Logging | `log/slog` (stdlib) | Structured JSON output |
| Containerisation | Docker | Single binary image |

---

## 4. Project Structure

```
.
├── cmd/
│   └── server/
│       └── main.go                  # Entry point: config, wiring, server start
├── internal/
│   ├── auth/
│   │   ├── cf_jwt.go                # CF Access JWT validation + claims extraction
│   │   ├── m2m_jwt.go               # M2M JWT validation + scope check
│   │   ├── middleware.go            # RequireCFAuth, RequireRole, RequireScope
│   │   └── role.go                  # Role type, group→role mapping
│   ├── device/
│   │   ├── model.go                 # Device struct
│   │   ├── repository.go            # Upsert, ListByClient, Delete
│   │   └── handler.go               # POST /api/v1/devices/register
│   ├── topic/
│   │   ├── model.go
│   │   ├── repository.go            # Create, List, Delete
│   │   └── handler.go               # GET|POST /topics, DELETE /topics/{slug}
│   ├── client/
│   │   ├── repository.go            # ListClients, AssignTopic, RemoveTopic
│   │   └── handler.go               # GET /clients, POST|DELETE /clients/{id}/topics/{slug}
│   ├── campaign/
│   │   ├── model.go
│   │   ├── repository.go            # Insert, UpdateStatus, List, SubscriberCount
│   │   ├── sender.go                # FCM send orchestration, stale token cleanup
│   │   └── handler.go               # GET /campaigns, GET /campaigns/compose,
│   ├── fcm/
│   │   └── client.go                # Firebase Admin SDK wrapper:
│   ├── db/
│   │   └── db.go                    # pgx pool initialisation
│   └── middleware/
│       └── common.go                # RequestID, structured request logging,
│                                    #   security headers
├── migrations/
│   ├── 001_create_topics.up.sql
│   ├── 001_create_topics.down.sql
│   ├── 002_create_devices.up.sql
│   ├── 002_create_devices.down.sql
│   ├── 003_create_client_topics.up.sql
│   ├── 003_create_client_topics.down.sql
│   └── 004_create_campaigns.up.sql
│   └── 004_create_campaigns.down.sql
├── templates/
│   ├── layout.html                  # Base layout, nav, env banner
│   ├── clients.html
│   ├── topics.html
│   └── campaigns/
│       ├── history.html
│       ├── compose.html             # Campaign compose form partial
│       └── confirm_modal.html       # Confirmation modal partial
├── static/
│   └── app.css
├── Dockerfile
├── docker-compose.yml               # Local dev only
└── .env.example
```

---

## 5. Configuration

All configuration is supplied via environment variables. No config files are committed.

```go
type Config struct {
    // Server
    Port            string `env:"PORT" envDefault:"8080"`
    Environment     string `env:"ENVIRONMENT" envDefault:"staging"` // "staging"|"production"

    // Database
    DatabaseURL     string `env:"DATABASE_URL,required"`

    // Firebase
    FirebaseCredentials string `env:"FIREBASE_CREDENTIALS_JSON,required"` // service account JSON

    // M2M JWT (device registration)
    M2MJWKSUrl      string `env:"M2M_JWKS_URL,required"`
    M2MAudience     string `env:"M2M_AUDIENCE,required"`

    // Cloudflare Access JWT (marketing UI)
    CFTeamDomain    string `env:"CF_TEAM_DOMAIN,required"`   // e.g. "yourteam.cloudflareaccess.com"
    CFAudience      string `env:"CF_AUDIENCE,required"`      // CF Application AUD tag

    // Role → Okta group mapping
    SenderGroup     string `env:"OKTA_SENDER_GROUP,required"`  // e.g. "marketing-senders"
    ViewerGroup     string `env:"OKTA_VIEWER_GROUP,required"`  // e.g. "marketing-viewers"
}
```

---

## 6. Database Schema

```sql
-- 001_create_topics
CREATE TABLE topics (
    slug        TEXT PRIMARY KEY,
    label       TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 002_create_devices
CREATE TABLE devices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       TEXT NOT NULL,
    fcm_token       TEXT NOT NULL,
    platform        TEXT,                   -- 'ios' | 'android' | 'web'
    app_version     TEXT,
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (client_id, fcm_token)
);
CREATE INDEX idx_devices_client_id ON devices(client_id);

-- 003_create_client_topics
CREATE TABLE client_topics (
    client_id   TEXT NOT NULL,
    topic_slug  TEXT NOT NULL REFERENCES topics(slug) ON DELETE CASCADE,
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    assigned_by TEXT NOT NULL,
    PRIMARY KEY (client_id, topic_slug)
);
CREATE INDEX idx_client_topics_topic_slug ON client_topics(topic_slug);

-- 004_create_campaigns
CREATE TABLE campaigns (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subject          TEXT NOT NULL,
    body             TEXT NOT NULL,
    link             TEXT,
    topic_slug       TEXT NOT NULL REFERENCES topics(slug),
    sent_by          TEXT NOT NULL,             -- Okta sub claim
    sent_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    fcm_message_id   TEXT,
    status           TEXT NOT NULL DEFAULT 'pending', -- pending|sent|failed
    error_message    TEXT,
    subscriber_count INT
);
CREATE INDEX idx_campaigns_sent_at ON campaigns(sent_at DESC);
```

**Migration path for `device_fingerprint`:** When a stable device ID is introduced by the trading app, add the following migration:

```sql
ALTER TABLE devices ADD COLUMN device_fingerprint TEXT;
CREATE UNIQUE INDEX idx_devices_fingerprint ON devices(device_fingerprint)
    WHERE device_fingerprint IS NOT NULL;
```

The upsert logic in `device.repository` will then be updated to prefer `device_fingerprint` as the conflict target when present.

---

## 7. HTTP Handlers & Routing

All routes are registered on `*http.ServeMux` in `main.go`. Middleware is applied via a chainable wrapper pattern.

```go
type Middleware func(http.Handler) http.Handler

func Chain(h http.Handler, m ...Middleware) http.Handler {
    for i := len(m) - 1; i >= 0; i-- {
        h = m[i](h)
    }
    return h
}
```

### Route Table

```
POST   /api/v1/devices/register         → device.Handler       [RequireM2MScope("register")]

GET    /clients                         → client.Handler        [RequireCFAuth, RequireRole(Viewer)]
POST   /clients/{client_id}/topics/{slug} → client.Handler     [RequireCFAuth, RequireRole(Sender)]
DELETE /clients/{client_id}/topics/{slug} → client.Handler     [RequireCFAuth, RequireRole(Sender)]

GET    /topics                          → topic.Handler         [RequireCFAuth, RequireRole(Viewer)]
POST   /topics                          → topic.Handler         [RequireCFAuth, RequireRole(Sender)]
DELETE /topics/{slug}                   → topic.Handler         [RequireCFAuth, RequireRole(Sender)]

GET    /campaigns                       → campaign.Handler      [RequireCFAuth, RequireRole(Viewer)]
GET    /campaigns/compose               → campaign.Handler      [RequireCFAuth, RequireRole(Sender)]
GET    /campaigns/confirm-modal         → campaign.Handler      [RequireCFAuth, RequireRole(Sender)]
POST   /campaigns                       → campaign.Handler      [RequireCFAuth, RequireRole(Sender)]

GET    /static/                         → http.FileServer       [no auth — static assets only]
```

Go 1.22 method-prefixed patterns (`"POST /path"`) are used throughout. No third-party router is needed.

---

## 8. Authentication & Authorisation

### 8.1 M2M JWT Middleware (`RequireM2MScope`)

```
Request
  └─ Extract "Authorization: Bearer <token>"
  └─ Parse JWT header → fetch JWKS (cached, 1h TTL via keyfunc)
  └─ Validate: signature, exp, aud == M2MAudience
  └─ Extract "scope" claim (string or []string)
  └─ Check scope contains required value → else 403
  └─ Call next handler
```

Token not present or invalid → `401 Unauthorized`.  
Token valid but scope missing → `403 Forbidden`.

### 8.2 Cloudflare Access JWT Middleware (`RequireCFAuth`)

Cloudflare Zero Trust injects `CF-Access-JWT-Assertion` on every proxied request. The middleware:

```
Request
  └─ Extract "CF-Access-JWT-Assertion" header → else 401
  └─ Fetch CF JWKS from https://<CFTeamDomain>/cdn-cgi/access/certs (cached, 1h TTL)
  └─ Validate: signature, exp, aud == CFAudience
  └─ Extract claims: sub (Okta user ID), email (for logging), groups []string
  └─ Map groups → Role (Sender | Viewer) → else 403
  └─ Store Role + sub in request context
  └─ Call next handler
```

### 8.3 Role Middleware (`RequireRole`)

```go
func RequireRole(minimum Role) Middleware {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            role := RoleFromContext(r.Context())
            if role < minimum {
                http.Error(w, "Forbidden", http.StatusForbidden)
                return
            }
            next.ServeHTTP(w, r)
        })
    }
}
```

`Role` is an `int` type where `Viewer = 1`, `Sender = 2`, enabling `>=` comparison.

The role is also passed into templates so the server conditionally renders sender-only controls:

```html
{{ if eq .Role "sender" }}
<button hx-post="/campaigns">Send Campaign</button>
{{ end }}
```

---

## 9. Device Registration Flow

**Endpoint:** `POST /api/v1/devices/register`

**Request body:**
```json
{
  "client_id":   "abc123",
  "fcm_token":   "<FCM registration token>",
  "platform":    "ios",
  "app_version": "2.4.1"
}
```

**Handler logic:**

```
1. Validate request body (client_id and fcm_token required)
2. Upsert into devices:
     INSERT INTO devices (client_id, fcm_token, platform, app_version, last_seen_at)
     VALUES ($1, $2, $3, $4, now())
     ON CONFLICT (client_id, fcm_token)
     DO UPDATE SET platform=EXCLUDED.platform,
                   app_version=EXCLUDED.app_version,
                   last_seen_at=now()
3. Query client_topics WHERE client_id = $1 → []topic_slug
4. For each topic_slug: fcm.SubscribeTokensToTopic([]string{fcm_token}, topic_slug)
   (best-effort; log errors but do not fail the request)
5. Return 200 OK
```

**Response (success):**
```json
{ "status": "ok" }
```

---

## 10. Campaign Send Flow

### 10.1 Compose → Confirm → Send

```
GET /campaigns/compose
  └─ Renders compose form partial (HTMX)
     Fields: subject (text), body (textarea), link (url), topic (select)

  [User submits form]
     hx-get="/campaigns/confirm-modal?subject=...&body=...&link=...&topic=crypto"
     hx-target="#modal-container"

GET /campaigns/confirm-modal?subject=&body=&link=&topic=
  └─ Queries: SELECT COUNT(DISTINCT client_id) FROM client_topics WHERE topic_slug=$1
  └─ Renders confirm_modal.html partial:
       "You are about to send to <N> clients on topic <label>. Confirm?"
       [Cancel] [Confirm → hx-post="/campaigns"]

POST /campaigns (form body: subject, body, link, topic)
  └─ Insert campaign row (status=pending)
  └─ Build FCM message (see §10.2)
  └─ Call fcm.Send(ctx, message)
     ├─ Success → UPDATE campaigns SET status='sent', fcm_message_id=$1
     └─ Failure → UPDATE campaigns SET status='failed', error_message=$1
  └─ Return HTMX partial: toast (success/error) + prepend row to history table
```

### 10.2 FCM Message Construction

```go
msg := &messaging.Message{
    Notification: &messaging.Notification{
        Title: campaign.Subject,
        Body:  campaign.Body,
    },
    Data: map[string]string{
        "link": campaign.Link,
    },
    Android: &messaging.AndroidConfig{
        Priority: "high",
    },
    APNS: &messaging.APNSConfig{
        Headers: map[string]string{
            "apns-priority": "10",
        },
    },
    Topic: campaign.TopicSlug,
}
```

The `link` is in the `Data` map so the client app can intercept it on tap and navigate without the OS handling it as a plain URL.

### 10.3 Stale Token Cleanup

FCM returns `messaging/registration-token-not-registered` when a token is expired or the app has been uninstalled. The `sender.go` cleanup is called after a failed send for individual tokens (relevant when topic membership changes trigger direct token operations):

```go
func (s *Sender) handleFCMError(ctx context.Context, token string, err error) {
    if messaging.IsRegistrationTokenNotRegistered(err) {
        _ = s.deviceRepo.DeleteByToken(ctx, token)
        // unsubscribe is implicit: FCM stops delivering to expired tokens,
        // but we also call UnsubscribeTokensFromTopic for cleanliness
        topics, _ := s.clientRepo.TopicsForToken(ctx, token)
        for _, t := range topics {
            _ = s.fcmClient.UnsubscribeTokensFromTopic(ctx, []string{token}, t)
        }
    }
}
```

---

## 11. Topic Assignment Flow

### Assign client to topic (`POST /clients/{client_id}/topics/{slug}`)

```
1. INSERT INTO client_topics (client_id, topic_slug, assigned_by)
   VALUES ($1, $2, $3)
   ON CONFLICT DO NOTHING
2. SELECT fcm_token FROM devices WHERE client_id = $1
3. fcm.SubscribeTokensToTopic(tokens, slug)   -- batch call
4. Return HTMX partial: updated topic badge list for the client row
```

### Remove client from topic (`DELETE /clients/{client_id}/topics/{slug}`)

```
1. DELETE FROM client_topics WHERE client_id=$1 AND topic_slug=$2
2. SELECT fcm_token FROM devices WHERE client_id = $1
3. fcm.UnsubscribeTokensFromTopic(tokens, slug)
4. Return HTMX partial: updated topic badge list
```

---

## 12. FCM Client Wrapper

```go
// internal/fcm/client.go

type Client struct {
    messaging *messaging.Client
}

func NewClient(ctx context.Context, credentialsJSON []byte) (*Client, error)

// Send sends a topic message. Returns FCM message ID.
func (c *Client) Send(ctx context.Context, msg *messaging.Message) (string, error)

// SubscribeTokensToTopic subscribes up to 1000 tokens per call (FCM limit).
// Automatically batches if len(tokens) > 1000.
func (c *Client) SubscribeTokensToTopic(ctx context.Context, tokens []string, topic string) error

// UnsubscribeTokensFromTopic unsubscribes tokens. Also batches at 1000.
func (c *Client) UnsubscribeTokensFromTopic(ctx context.Context, tokens []string, topic string) error
```

FCM's `SubscribeToTopic` / `UnsubscribeFromTopic` batch APIs accept up to **1000 tokens per call**. The wrapper handles batching transparently.

---

## 13. HTMX UI Patterns

The UI uses HTMX attributes exclusively. No custom JavaScript is written unless strictly necessary (e.g. modal close on backdrop click).

### Partial Swap Pattern

Every mutating action returns a minimal HTML partial that replaces a target element:

| Action | `hx-target` | Swap |
|---|---|---|
| Assign topic | `#client-{id}-topics` | `innerHTML` |
| Remove topic | `#client-{id}-topics` | `innerHTML` |
| Open confirm modal | `#modal-container` | `innerHTML` |
| Campaign sent | `#toast-container`, `#campaign-tbody` | `innerHTML`, `afterbegin` |
| Create topic | `#topics-table` | `outerHTML` |

### Environment Banner

The layout template receives `Env` from context. When `Env != "production"`:

```html
{{ if ne .Env "production" }}
<div class="env-banner">⚠ STAGING ENVIRONMENT</div>
{{ end }}
```

---

## 14. Logging

All logs are emitted as structured JSON via `log/slog` to stdout. Each log entry includes:

| Field | Source |
|---|---|
| `time` | UTC RFC3339 |
| `level` | INFO / WARN / ERROR |
| `msg` | Human-readable description |
| `request_id` | UUID generated per request |
| `method` | HTTP method |
| `path` | Request path |
| `status` | Response HTTP status code |
| `duration_ms` | Handler duration |
| `actor` | Okta sub (UI requests) or `"m2m"` (API requests) |

Request logging middleware writes one log line per request on completion.

---

## 15. Deployment

### Dockerfile

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o server ./cmd/server

FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/server /server
COPY --from=builder /app/templates /templates
COPY --from=builder /app/static /static
ENTRYPOINT ["/server"]
```

### Environment Variables (per environment)

```
PORT=8080
ENVIRONMENT=production

DATABASE_URL=postgresql://...supabase...

FIREBASE_CREDENTIALS_JSON=<service account JSON, single line>

M2M_JWKS_URL=https://your-oauth2-provider/.well-known/jwks.json
M2M_AUDIENCE=push-notification-service

CF_TEAM_DOMAIN=yourteam.cloudflareaccess.com
CF_AUDIENCE=<CF Application AUD tag>

OKTA_SENDER_GROUP=marketing-senders
OKTA_VIEWER_GROUP=marketing-viewers
```

### Infrastructure

Each environment runs:
- 1× Go service container
- 1× Supabase project (direct Postgres connection via `DATABASE_URL`)
- 1× Firebase project (separate service account per environment)
- Cloudflare Zero Trust application (separate per environment, separate AUD)

---

## 16. Open Items

| Item | Notes |
|---|---|
| Stable `device_id` | Schema has migration path ready. Implement when trading app defines the identifier. |
| Campaign scheduling | `campaigns` table will need a `scheduled_for TIMESTAMPTZ` column. Introduce Asynq + Redis for job execution. |
| FCM topic send error granularity | Topic sends return a single message ID, not per-device status. Per-device tracking would require switching to manual fan-out. |
| Delivery metrics | Open rate / click-through tracking not in scope for v1. Would require a link-shortening/redirect service and client-side instrumentation. |
