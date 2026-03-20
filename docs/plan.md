# Marketing Notification Service — Development Plan

---

## Phase 1: Project Foundation

1. **Set up GitHub repository**
   - [x] Monorepo structure: `/backend`, `/frontend`, `/docs`
   - [x] Add `.gitignore`, `README.md`, and `LICENSE`
   - [x] Set up branch protection rules and PR templates

2. **Set up project infrastructure**
   - Provision Supabase project (dev + prod environments)
   - Set up Firebase project and generate FCM service account key
   - Configure Cloudflare Zero Trust application and access policies for the frontend and backend
   - Set up environment variable management (`.env.example` files per service)

3. **Set up CI/CD pipeline**
   - GitHub Actions workflows for backend (lint, test, build) and frontend (lint, test, build)
   - Docker image build and push for the backend
   - Deployment pipeline (e.g. to Cloud Run, Railway, Fly.io, or similar)

---

## Phase 2: Data Modeling

4. **Define database schemas**
   - `users` — internal representation of marketing team members (linked to IdP identity, e.g. by email or external ID)
   - `device_tokens` — stores FCM tokens per end user (token, platform, created/updated timestamps, active flag)
   - `topics` — available subscription topics (name, slug, description)
   - `subscriptions` — join table between device tokens and topics
   - `notifications` — log of sent notifications (title, body, topic or target, status, sent_at)
   - `notification_targets` — optional: individual token targeting in addition to topic-based sends

5. **Define Row Level Security (RLS) policies in Supabase**
   - Since the backend service will own all DB access (via service role key), define RLS to block direct client access
   - Document access patterns per table

6. **Set up database migrations tooling**
   - Choose and configure a migration tool (e.g. `golang-migrate` or `goose`)
   - Write initial migrations from the schemas defined above
   - Apply migrations to dev environment and verify

---

## Phase 3: Backend Service

7. **Scaffold the Go service**
   - Define project structure (`/cmd`, `/internal`, `/pkg`, `/api`)
   - Set up HTTP router (e.g. `chi` or `gin`)
   - Set up config loading (env vars), logging, and graceful shutdown

8. **Implement Cloudflare Access token verification middleware**
   - Fetch and cache Cloudflare Access public keys (JWKS)
   - Validate JWT signature, expiry, and audience on each request
   - Extract group claims from the JWT for authorization (e.g. `marketing`, `developer`)
   - Return `401`/`403` appropriately

9. **Define and document the API contract**
   - Write an OpenAPI (Swagger) spec covering all endpoints before implementation
   - Endpoints to cover: device token registration/deregistration, topic CRUD, subscription management, notification sending, notification history

10. **Implement device token management endpoints**
    - Register a device token (create or upsert)
    - Deactivate / delete a device token
    - List tokens (filterable by topic, platform, active status)

11. **Implement topic management endpoints**
    - Create, update, delete topics
    - List topics with subscriber counts

12. **Implement subscription management endpoints**
    - Subscribe a token to a topic
    - Unsubscribe a token from a topic
    - List subscriptions for a token

13. **Implement FCM integration**
    - Initialize the Firebase Admin SDK with the service account key
    - Implement topic-based notification sending (FCM `SendToTopic`)
    - Implement token-based (direct) notification sending as a fallback
    - Handle FCM error responses (invalid token, unregistered device — deactivate tokens accordingly)

14. **Implement notification sending endpoint and history logging**
    - Accept notification payload (title, body, data, target topic or token list)
    - Dispatch to FCM
    - Log result to the `notifications` table (success/failure, FCM message ID)

15. **Write backend tests**
    - Unit tests for middleware, FCM error handling, and business logic
    - Integration tests using **Testcontainers for Go**:
      - Spin up a real Postgres container per test suite
      - Run migrations against it before tests execute
      - Test full request→DB roundtrips for all key endpoints
      - Use a mock FCM HTTP server (e.g. `httptest`) to simulate FCM responses without real credentials

---

## Phase 4: Frontend Dashboard

16. **Scaffold the React + Vite frontend**
    - Set up Vite project with TypeScript
    - Add routing (e.g. `react-router-dom`), state management, and an HTTP client (e.g. `axios` or `ky`)
    - Configure Cloudflare Zero Trust — the app will be behind CF Access so no login UI is needed; rely on the injected identity

17. **Build device token management UI**
    - Table view of registered tokens (filterable/searchable)
    - Ability to deactivate or delete tokens

18. **Build topic management UI**
    - CRUD interface for topics
    - View subscriber count per topic

19. **Build notification composer UI**
    - Form to compose a notification (title, body, optional data payload)
    - Target selector: choose a topic or specific tokens
    - Preview and send

20. **Build notification history UI**
    - Table of past notifications with status, target, and timestamp
    - Basic filtering by date range and topic

21. **Write frontend tests**
    - Component tests for the composer and token table
    - Basic end-to-end test for the send notification happy path (e.g. with Playwright)

---

## Phase 5: Hardening & Developer Experience

22. **Set up local development environment with Docker Compose**
    - Define a `docker-compose.yml` at the repo root covering:
      - Postgres (mirroring Supabase's Postgres version)
      - The Go backend service (with hot-reload via `air`)
      - The Vite frontend (with HMR via the dev server)
      - A **fake FCM server** (a lightweight HTTP mock) to simulate push delivery locally without real Firebase credentials
    - Add a `docker-compose.override.yml` pattern for developer-specific overrides
    - Provide a `make dev` (or similar) shortcut to bring the full stack up in one command
    - Document the local setup in `README.md`, including how to seed the DB and how to swap in a real FCM key for local end-to-end testing

23. **Expose and document the developer API**
    - Publish the OpenAPI spec (e.g. Swagger UI or Redoc hosted on the service)
    - Document authentication requirements for developers (CF Access token flow)
    - Provide example `curl` / SDK usage

23. **Add observability**
    - Structured logging on the backend (e.g. `slog` or `zerolog`)
    - Request tracing (correlation IDs)
    - Error alerting (e.g. Sentry or a simple webhook)

24. **Security review**
    - Verify CF Access JWT validation is not bypassable
    - Confirm group claim enforcement is applied to all mutating endpoints
    - Ensure the FCM service account key is never logged or exposed

25. **Staging deployment and QA**
    - Deploy the full stack to a staging environment
    - End-to-end test with a real FCM device token
    - Marketing team UAT (user acceptance testing)

26. **Production deployment and monitoring**
    - Apply DB migrations to production
    - Deploy backend and frontend
    - Set up uptime monitoring and alert thresholds
