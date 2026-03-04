# Product Requirements Document
## Push Notification Broadcasting Service

**Version:** 1.0  
**Status:** Draft  
**Last Updated:** 2026-03-04

---

## 1. Purpose

This document defines the product requirements for a push notification broadcasting service that enables the marketing team to send targeted push notifications to trading platform clients via Firebase Cloud Messaging (FCM). The service provides a web UI for topic management, client assignment, and campaign delivery.

---

## 2. Background & Motivation

The marketing team needs a self-serve tool to broadcast push notifications to segmented groups of trading platform clients (e.g. clients interested in crypto, fiat, options, CFD). Currently there is no internal tooling for this. The service must integrate with the trading platform's existing client infrastructure and deliver notifications reliably across all of a client's registered devices.

---

## 3. Goals

- Enable the marketing team to send push notifications to topic-based client segments without engineering involvement.
- Allow topic-based segmentation and client assignment to be managed entirely through the UI.
- Ensure every registered device of a target client receives the notification.
- Maintain a full audit trail of all campaigns sent.
- Keep the system free of PII data.
- Support two isolated environments: staging and production.

## 4. Non-Goals (v1)

- Scheduled / delayed campaign delivery (deferred to v2).
- Per-device delivery receipts or read tracking.
- Rich media notifications (images, actions).
- A/B testing of notification copy.
- Client self-service topic subscription from within the trading app.
- Email or SMS fallback channels.

---

## 5. Users & Roles

| Role | Description | Permissions |
|---|---|---|
| **Viewer** | Marketing team member who monitors campaigns | View campaign history, client list, topic list |
| **Sender** | Marketing team member who runs campaigns | All Viewer permissions + create/send campaigns, manage topics, assign clients to topics |

Roles are derived from Okta group membership, surfaced via the `groups` claim in the Cloudflare Access JWT. Role enforcement is server-side.

---

## 6. Functional Requirements

### 6.1 Device Registration

| ID | Requirement |
|---|---|
| FR-01 | The service shall expose a public HTTP endpoint `POST /api/v1/devices/register` for the trading app to register device FCM tokens. |
| FR-02 | The registration endpoint shall accept `client_id`, `fcm_token`, `platform` (ios/android/web), and `app_version`. |
| FR-03 | The endpoint shall be protected by a machine-to-machine OAuth2 Client Credentials JWT. The JWT must contain a `scope` claim with value `register`. Requests without a valid token or correct scope shall be rejected with HTTP 403. |
| FR-04 | A client may register multiple devices. Each `(client_id, fcm_token)` pair is treated as a unique device record. |
| FR-05 | If the same `(client_id, fcm_token)` pair is re-registered, the record shall be upserted: `last_seen_at`, `platform`, and `app_version` are updated; no duplicate row is created. |
| FR-06 | Upon successful registration, if the client is already assigned to one or more topics, the new FCM token shall be subscribed to those topics in FCM immediately. |
| FR-07 | Stale device tokens reported by FCM (error: `registration-token-not-registered`) shall be automatically deleted from the database and unsubscribed from all FCM topics. |

### 6.2 Topic Management

| ID | Requirement |
|---|---|
| FR-08 | Senders shall be able to create a topic with a `slug` (e.g. `crypto`) and a human-readable `label` (e.g. `Cryptocurrency`). |
| FR-09 | Senders shall be able to delete a topic. Deleting a topic shall also remove all `client_topics` assignments for that topic. |
| FR-10 | All users shall be able to view the list of topics. |
| FR-11 | Topic slugs shall be unique, URL-safe, and immutable after creation. |

### 6.3 Client–Topic Assignment

| ID | Requirement |
|---|---|
| FR-12 | Senders shall be able to assign a client to a topic via the UI by specifying `client_id`. |
| FR-13 | When a client is assigned to a topic, all of that client's registered FCM tokens shall be subscribed to the FCM topic immediately. |
| FR-14 | Senders shall be able to remove a client from a topic. All of that client's FCM tokens shall be unsubscribed from the FCM topic immediately. |
| FR-15 | The client list view shall display each client's `client_id`, their assigned topics, and number of registered devices. |
| FR-16 | The client list shall be searchable by `client_id`. |

### 6.4 Campaign Composition & Sending

| ID | Requirement |
|---|---|
| FR-17 | Senders shall be able to compose a campaign with: **Subject** (notification title), **Body** (notification body), **Link** (deep-link URL), and **Topic** (target segment). |
| FR-18 | Before sending, the UI shall display a confirmation modal showing the target topic and the number of distinct clients subscribed to it. |
| FR-19 | Senders must explicitly confirm before the campaign is dispatched. |
| FR-20 | On confirmation, the service shall send a single FCM topic message. FCM handles fan-out to all subscribed tokens. |
| FR-21 | The `link` shall be delivered as an FCM data payload key, not inside the notification block, so the client app can handle deep-linking. |
| FR-22 | Each campaign shall be recorded in the database with: subject, body, link, topic, sender identity (Okta sub), timestamp, FCM message ID, status (`pending` → `sent` / `failed`), and subscriber count snapshot. |
| FR-23 | If FCM returns an error, the campaign record shall be updated to `failed` with the error message stored. The UI shall display an error to the sender. |

### 6.5 Campaign History

| ID | Requirement |
|---|---|
| FR-24 | All users shall be able to view a paginated campaign history table showing: subject, topic, sender, timestamp, status, and subscriber count. |
| FR-25 | Campaign history shall be read-only. Sent campaigns cannot be edited or deleted. |

### 6.6 Dry-Run / Staging

| ID | Requirement |
|---|---|
| FR-26 | A separate staging environment shall be deployed with its own Supabase database and Firebase project. |
| FR-27 | The staging environment accepts device registrations from staging trading app builds only. |
| FR-28 | The marketing UI in staging shall be visually distinct (e.g. environment banner) to prevent accidental production sends. |

---

## 7. Non-Functional Requirements

| ID | Requirement |
|---|---|
| NFR-01 | The service shall store no PII. `client_id` is an opaque identifier sourced from the trading platform. |
| NFR-02 | The device registration endpoint shall handle bursts of registrations (e.g. app release causing many simultaneous re-registrations) without data corruption. Upserts shall be atomic. |
| NFR-03 | The marketing UI shall be accessible only via Cloudflare Zero Trust. Direct access to the UI without a valid CF Access session shall return HTTP 401. |
| NFR-04 | Role enforcement (viewer vs sender) shall be applied server-side. UI elements may be conditionally hidden, but access control must not rely solely on UI suppression. |
| NFR-05 | JWKS keys for both the M2M JWT and CF Access JWT shall be cached with a configurable TTL to avoid fetching on every request. |
| NFR-06 | The service shall be deployable as a single Docker container. Environment-specific configuration shall be supplied via environment variables. |
| NFR-07 | All database operations shall use parameterised queries. No raw string interpolation into SQL. |
| NFR-08 | The service shall emit structured JSON logs including: request ID, endpoint, status code, duration, and actor identity (for UI requests). |

---

## 8. User Interface Requirements

| ID | Requirement |
|---|---|
| UI-01 | The UI shall be server-rendered using Go `html/template` and HTMX. No SPA framework. |
| UI-02 | Page interactions (topic assignment, campaign sending, search) shall use HTMX partial swaps without full page reloads. |
| UI-03 | The confirmation modal (FR-18) shall be an HTMX partial rendered server-side and injected into the page. |
| UI-04 | Viewer-only users shall not see the campaign compose form, send button, topic create/delete controls, or client assignment controls. These shall be absent from the server-rendered HTML entirely, not merely hidden. |
| UI-05 | The staging environment UI shall display a persistent environment indicator (e.g. "STAGING" banner) on all pages. |
| UI-06 | The UI shall display a success or error toast after a campaign send attempt. |

---

## 9. Security Requirements

| ID | Requirement |
|---|---|
| SEC-01 | The device registration endpoint (`/api/v1/devices/register`) shall validate the M2M JWT: signature (via JWKS), expiry, audience, and `scope=register`. |
| SEC-02 | All marketing UI routes shall validate the Cloudflare Access JWT on every request via the CF JWKS endpoint. |
| SEC-03 | The `groups` claim extracted from the CF JWT shall be used exclusively to determine the user's role. Groups-to-role mapping shall be configurable via environment variables. |
| SEC-04 | HTTP responses for the UI shall include standard security headers: `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`. |
| SEC-05 | The Firebase service account credentials and database connection string shall never be committed to source control. They shall be injected via environment variables or a secrets manager. |

---

## 10. Out of Scope for v1

- Delivery receipts / open rate tracking
- Campaign scheduling
- Notification templates
- Multi-language / localisation support
- Self-service client topic subscription from the trading app
- Admin UI for managing marketing team user accounts (handled via Okta)

---

## 11. Success Criteria

- The marketing team can send a campaign to a topic within 2 minutes of deciding to do so, without engineering assistance.
- Device registration handles concurrent upserts correctly with no duplicate rows.
- All campaigns are recorded in history with full audit metadata.
- No PII is present in the database.
- Staging and production are fully isolated with no shared Firebase project or database.
