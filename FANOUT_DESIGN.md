# Fanout - Design and Architecture Document

> Working name: **Fanout**
> Status: Pre-development, architecture defined
> Revision: 2 hosting tiers verified, data model revised to event-sourced, delivery guarantees specified, hosting made provider-independent
> Last updated: 10 October 2026

---

## 1. What Is Fanout

Fanout is a community sports live score broadcasting platform for India. It fills the gap that Google's sports panel and professional data providers leave for non-professional sports: school tournaments, college leagues, community cricket, amateur football, kabaddi, and any other local competition that will never have an official data feed.

Organisers broadcast live scores from their phone during a match. Audiences follow in real time from anywhere, without needing to sign up or install anything. The experience mirrors what Google does for the IPL or the Premier League, but for the match happening at the ground two streets away.

---

## 2. Core Principles

- **Audience viewing is free forever.** Monetisation flows through organisers and businesses, never viewers.
- **Clarity, Speed, Trust.** Every screen has one primary action. Organisers are managing a live match. Audiences are checking a score mid-conversation. Neither has time to think.
- **No friction to value.** A shared event link shows the live score within two taps, with no sign-up wall.
- **Correctness over throughput.** A wrong score destroys the trust the entire platform depends on. Every write path is idempotent and every read path can detect its own staleness.
- **Surgical AI usage.** AI moderation API calls are reserved for event descriptions, profile names, and reported content only. Structured score data never touches a moderation API.
- **Platform-agnostic code.** Every external dependency sits behind an environment variable and a Java interface. Migrating from one provider to another is a config change, not a code project.
- **The architecture does not depend on any provider's free tier.** Free tiers are an opportunistic cost optimisation, never a design input. The system is specified against a resource requirement (section 10) that many providers can satisfy.
- **Independent scalability.** Read and write infrastructure scale separately. WebSocket connections never share a process with business logic.

---

## 3. Platforms

| Surface | Technology |
|---|---|
| Web app | React + Vite |
| Android app | Capacitor wrapping the React web app |
| iOS app | Capacitor wrapping the React web app |

Electron.js desktop has been dropped. Flutter has been dropped in favour of Capacitor, which allows one React codebase to ship to web and both mobile platforms without maintaining a separate codebase.

iOS note: building and submitting the iOS app requires macOS hardware and an Apple Developer account (USD 99/year). Android ships first; iOS follows once the web and Android surfaces are stable.

---

## 4. User Roles

### Audience / Viewer
- No sign-up required to view any event, ever
- Discovers events by location and sport type
- Receives live score updates via WebSocket
- Can flag inappropriate content (three weighted flags auto-hides)
- Sign-up is prompted gently after three or more minutes of watching, never forced
- Sign-up unlocks: following organisers, push notifications for followed events

### Event Organiser
- Registers with OTP verification (phone or email)
- Creates and manages events with a sport-specific quick-tap scoring interface
- Can invite co-managers who manage the same event simultaneously
- Co-manager access is approved by the event creator
- New organisers enter a probationary period: first two events require admin approval before going public
- Builds a trust score over time; high-trust organisers bypass the approval queue

### Co-Manager
- Invited by the event creator
- Has identical scoring permissions during the event
- Co-manager presence is shown to all active managers with a live indicator
- Concurrent submissions are ordered by the append-only score event log (section 6.2); the losing write is surfaced as a conflict notice and both entries remain permanently in the audit trail

### Admin
- Internal role only
- Desktop-primary, information-dense layout
- Manages the moderation queue, organiser approvals, content flags, and account actions
- Every action is logged in a permanent, append-only audit trail
- Action labels are always descriptive ("Approve and Publish", "Ban Account"), never ambiguous ("OK", "Confirm")
- Destructive permanent actions require the admin to type the target account name to confirm

### Trust Score (definition)

Referenced throughout revision 1 but never defined. Specified here because it gates the approval queue and flag weighting.

Score starts at 0, range 0–100.

| Signal | Delta |
|---|---|
| Event completed with >= 10 score updates and a final result | +8 |
| Event completed with a co-manager present | +3 |
| OTP-verified contact retained 30 days | +5 (once) |
| Verified Organiser badge purchased, identity check passed | +25 (once) |
| Event auto-flagged by sham detection | -15 |
| Content upheld against this account after flag review | -20 |
| Admin warning issued | -25 |
| Event abandoned (marked live, never completed) | -10 |

Thresholds:
- `< 20` - **probation**: admin approval required before an event publishes
- `20–59` - **normal**: auto-publish, standard rate limits
- `>= 60` - **trusted**: auto-publish, relaxed rate limits, flags raised against this account need two reports before auto-hide

Flag weight when this user reports content: `1 + floor(trust / 40)`, capped at 3.

---

## 5. Feature Scope by Phase

### Phase 1 - Foundation

Sports supported: Cricket (simplified model, see below), Football, Basketball, plus a generic fallback panel.

**Organiser features**
- OTP registration and probationary approval flow
- Create events: sport, teams, venue, format, date/time
- Sport-specific quick-tap scoring interface
- Co-manager invite and conflict detection
- 5-second undo toast after every score entry
- **Offline-tolerant score entry**: taps queue locally and flush on reconnect (section 7.6). Promoted from Phase 3 - poor connectivity describes most Indian grounds, and a lost tap is a broken core promise.
- Mark event live, pause, and complete
- Share event link (public URL, no app required to view)

**Audience features**
- Location-based event discovery (live and upcoming)
- Filter by sport and distance
- Live score panel via WebSocket, with gap detection and automatic resync
- Community flagging
- Share event link

**Moderation**
- Local Java profanity filter on all text fields
- Organiser probation (first two events require approval)
- Weighted three-flag auto-hide
- Sham detection: no score update within 45 minutes of going live triggers an admin flag
- Admin dashboard: approve, reject, warn, suspend, ban

**Monetisation**: none. Free tier only.

**Phase 1 cricket scope is deliberately narrow.** Full cricket scoring is a genuinely hard product problem - wides, no-balls, free hits, byes versus leg-byes, strike rotation on odd runs, over completion, retired hurt, declarations. A half-correct cricket scorer produces wrong scorecards, and wrong scores destroy platform trust faster than any missing feature. Phase 1 ships:

- Runs (0–6 taps), wickets, overs bowled, extras as a single undifferentiated counter
- Per-innings totals, current run rate, target
- No per-ball extras taxonomy, no strike rotation, no individual batter or bowler cards

The scoring UI states "Simplified scoring" explicitly. Full ball-by-ball cricket is a Phase 2 deliverable with dedicated design time. Shipping simplified-and-correct beats shipping detailed-and-wrong.

### Phase 2 - Stickiness

Additional sports: Kabaddi, Volleyball, Badminton, Tennis, Throwball.

- Full ball-by-ball cricket model (extras taxonomy, strike rotation, batter and bowler cards)
- Image uploads: event banner, team photo, post-match highlights (async moderation pipeline)
- Push notifications for followed events and organisers
- Post-match summary auto-generation
- PDF and CSV export of match stats
- Sport config registry served from the database: JSON-driven schema per sport, dynamic UI generation, no app release needed for a new sport
- Paid tiers via Razorpay (India) and Stripe (international)
- Verified Organiser badge (one-time fee, identity check)

### Phase 3 - Ecosystem

- Full season and league management: fixtures, standings, playoffs, brackets
- Player profiles with cross-event career statistics
- Video reel uploads (60–90 seconds, paid tiers only, frame-sampled moderation)
- Embed widget: live score widget for an organiser's own site or WhatsApp channel
- Audience engagement: polls, MVP voting, reactions
- Sponsor slot on event pages (organiser sells it, Fanout enables display)
- Searchable event archive (free tier limited to last 3 months)
- Community-submitted sport configs: organisers define new sports, admin reviews and publishes

---

## 6. Data Architecture

### 6.1 Event Model (Two-Layer)

Every event has a universal set of fields regardless of sport:

```
Event
  id, sport_type, status (upcoming | live | paused | completed)
  home_team, away_team
  venue_name, venue_location (PostGIS point)
  start_time, end_time
  current_period_label
  home_score, away_score
  organiser_id
  seq                  -- monotonic revision counter, see 6.3
  event_timeline[]     -- ordered log of key moments
  sport_data (JSONB)   -- sport-specific stat block (DERIVED, see 6.2)
```

The `sport_data` JSONB column holds sport-specific state. Cricket stores innings, overs and totals. Football stores goals, cards and substitutions. Adding a new sport requires no schema migration, only a new entry in the sport config registry.

This two-layer split is the strongest decision in revision 1 and is carried forward unchanged.

### 6.2 The Score Event Log Is Authoritative

**Changed from revision 1.** Revision 1 treated `sport_data` as mutable source of truth, with a score log as a side table. That is inverted here.

```
ScoreEvent  (append-only; never updated, never deleted)
  id (uuid)
  event_id
  seq                 -- monotonic per event_id, gap-free, assigned on commit
  client_event_id     -- uuid generated on the device; UNIQUE per event_id
  action_type         -- from the sport config's quick_actions
  payload (JSONB)
  actor_id            -- organiser or co-manager
  created_at
  superseded_by       -- nullable; set when a compensating event voids this one
```

`events.sport_data` and the denormalised `home_score` / `away_score` columns are a **projection** of this log, updated in the same transaction as the append, and rebuildable from scratch at any time.

This single inversion delivers four features the design already requires, with no additional mechanism:

| Requirement | How the log provides it |
|---|---|
| 5-second undo | Append a compensating event, set `superseded_by` on the target. No mutation, no lost history. |
| Co-manager conflict resolution | `seq` assignment at commit is the total order. Last write wins deterministically; the losing entry stays visible in the audit trail. |
| Permanent audit log | The log *is* the audit log. No separate write, no drift between them. |
| Phase 3 Kafka replay | The log is already an event stream. Migration becomes a producer swap, not a redesign. |

**Idempotency.** `client_event_id` is generated on the device before the request leaves it, with `UNIQUE (event_id, client_event_id)` enforced in Postgres. An organiser who taps "4 runs", sees a timeout on stadium Wi-Fi, and taps again produces one score event, not eight runs. On a duplicate key the server returns the existing event rather than an error, so the retry looks like success to the client.

Without this, double-counted taps on flaky networks are the single most likely source of the first wave of "the score is wrong" reports.

### 6.3 Revision Sequencing and Delta Integrity

**New in revision 2.** Revision 1 specified full state on page open, then deltas forever. Any dropped frame, reconnect, or brief network blip left that client permanently and silently wrong, with no way to detect it.

Every event carries a monotonic `seq`. Every delta pushed over the WebSocket carries both `seq` and `prev_seq`.

```
Client holds last_seq.
On receiving a delta:
  delta.prev_seq == last_seq  -> apply; last_seq = delta.seq
  delta.seq      <= last_seq  -> discard (duplicate or reorder)
  delta.prev_seq >  last_seq  -> GAP: discard, refetch full state, resume
```

A gap is now detectable and self-healing; silent divergence becomes impossible. The full-state refetch is a single cached Redis read, so resync is cheap enough to do liberally rather than reluctantly.

### 6.4 Sport Config Registry

Each sport is defined as a JSON config:

```json
{
  "sport": "kabaddi",
  "period_label": "Half",
  "periods": 2,
  "scoring_unit": "points",
  "quick_actions": ["Raid Point", "Tackle Point", "Super Raid", "All Out"],
  "stat_display_blocks": ["half_scores", "raid_stats", "tackle_stats"],
  "max_updates_per_minute": 10
}
```

The React frontend reads this config to render the quick-tap scoring interface and the display panel dynamically. No frontend code change is needed to support a new sport once its config is published.

Phase 1 ships configs as versioned files generated into both the frontend and the Event Service from one shared JSON source. Phase 2 moves them into the database and serves them over the API with an ETag, enabling new sports without an app release.

The config version is recorded on every score event, so historical events always render under the schema they were scored with.

### 6.5 Database Strategy

Each microservice owns its own PostgreSQL schema and connects with a database role granted on that schema alone. No service holds credentials capable of reading another service's tables. Cross-schema access is blocked **at the database**, not by convention - so rule 6 in section 13 is enforced rather than merely documented.

| Service | Schema contents |
|---|---|
| Identity Service | Users, OTP records, refresh tokens, roles, trust score ledger |
| Event Service | Events, sport configs, team records |
| Scoring Service | Score event log (authoritative), projections, hot state (also in Redis) |
| Media Service | Upload records, moderation state |
| Moderation Service | Moderation queue, decisions, audit log |
| Notification Service | Device tokens, notification delivery log |
| Admin Service | Audit trail; read-only grants on other schemas (see below) |

**Admin Service read access.** Revision 1 specified a read replica. Verified 10 Oct 2026: Supabase's free plan provides no read replicas, and a single self-hosted Postgres has none either. Phase 1 therefore grants the Admin Service a dedicated read-only role with `SELECT` on the specific **views** each service exposes for it - never raw tables. This preserves the isolation intent, keeps the dependency explicit and versioned, and becomes a genuine read replica at the managed-database migration step with no application change.

**Connection budget.** Eleven services against one Postgres is a real constraint, and the default would break it. HikariCP defaults to a pool of 10 per service; eleven services would want 110 connections. Supabase Nano and Micro both cap at 60 direct connections and 200 pooler clients (verified 10 Oct 2026), and a self-hosted Postgres on a small host should not be configured much higher.

Policy: every service sets `maximum-pool-size: 4`, `minimum-idle: 1`, and connects through a transaction-mode pooler (PgBouncer when self-hosted, Supavisor on Supabase). Total budget 44, leaving headroom for migrations and admin sessions.

Transaction-mode pooling disables server-side prepared statements. Set `prepareThreshold=0` on the JDBC URL and be aware this will otherwise surprise you in JPA.

---

## 7. Backend Architecture

Fanout uses a Spring Microservices architecture with Spring Cloud. The service boundaries exist naturally in the domain and are not artificial.

### 7.1 Services

**fanout-gateway**
Spring Cloud Gateway. Single entry point for all client traffic. Handles CORS, rate limiting, and JWT validation via a global filter. Routes to downstream services by URL pattern, resolving addresses via Eureka (no hardcoded URLs). Injects a correlation ID into every request. Clients never talk to individual services directly.

```
/api/auth/**        -> identity-service
/api/events/**      -> event-service
/api/scoring/**     -> scoring-service
/api/media/**       -> media-service
/api/admin/**       -> admin-service   (admin role required)
/ws/**              -> websocket-service
```

**fanout-eureka-server**
Netflix Eureka service registry. Every service registers on startup. The gateway resolves `lb://service-name` to a live instance, enabling horizontal scaling of individual services with no config change.

**fanout-config-server**
Spring Cloud Config Server. All services pull configuration at startup, backed by a Git repository. Shared values (Redis URL, JWT secret, object storage credentials) are defined once. Secrets are supplied as environment variables to the Config Server itself and never committed.

**fanout-identity-service**
Owns all authentication and authorisation: OTP generation and verification, JWT issuance and refresh, role management, trust score ledger. Every other service trusts the JWT the gateway has already validated and does not re-authenticate requests arriving from the gateway.

**fanout-event-service**
Core domain service. Event CRUD, sport config registry, event lifecycle, team management. The heaviest domain logic. Reads live state from the Scoring Service's Redis projection, never from its tables.

**fanout-scoring-service**
Handles live score writes exclusively. Validates input against the sport config, appends to the score event log, updates the projection, refreshes the Redis hot cache, and publishes a delta to Redis pub/sub - the publish occurring after transaction commit. Separated from the Event Service because write frequency during a live match is orders of magnitude higher and must scale independently.

**fanout-websocket-service**
Its only responsibility is holding open WebSocket connections and pushing score deltas to audience clients. Subscribes to Redis pub/sub. Stateless except for the active connection registry. Scales horizontally with the number of concurrent live events, independently of everything else.

**fanout-media-service**
Generates presigned upload URLs for S3-compatible object storage. Receives upload completion callbacks. Triggers the Moderation Service via RabbitMQ. Manages thumbnail generation. No business logic, purely I/O coordination.

**fanout-moderation-service**
Async only. Consumes moderation tasks from RabbitMQ, runs the local profanity filter, calls the configured image moderation provider, applies frame sampling for video. Publishes results back to the queue. Never in the synchronous request path.

**fanout-notification-service**
Consumes events from RabbitMQ (event going live, final result, co-manager invite accepted, followed-organiser activity) and dispatches Firebase FCM messages to registered device tokens. Fire-and-forget, async, never in the critical path.

**fanout-admin-service**
Serves the admin dashboard backend only. Read-only grants on other services' exposed views. Writes to the audit log. Issues bans, suspensions and warnings. Isolated so admin traffic never competes with public traffic and can carry stricter network access controls.

### 7.2 Spring Cloud Components

| Component | Role |
|---|---|
| Spring Cloud Gateway | API gateway, JWT filter, routing, rate limiting, correlation ID injection |
| Netflix Eureka | Service registry and discovery |
| Spring Cloud Config | Centralised configuration server |
| Spring Cloud OpenFeign | Declarative HTTP client for synchronous inter-service calls |
| Resilience4j | Circuit breaker, retry, rate limiter, bulkhead on Feign clients |
| Micrometer + Prometheus | Metrics from every service |
| Spring Cloud Bus (Phase 2) | Live config change propagation via RabbitMQ |

### 7.3 Message Broker

RabbitMQ, self-hosted alongside the services in Phase 1. Used for all async inter-service communication:

- Scoring Service publishes score milestone events, consumed by Notification Service
- Media Service publishes upload-complete events, consumed by Moderation Service
- Moderation Service publishes results, consumed by Media Service and Event Service
- Event Service publishes event-live and event-complete events, consumed by Notification Service

Every consumer is idempotent, keyed on the message's event id. Every queue has a dead-letter exchange. Nothing retries indefinitely.

Self-hosting is deliberate: CloudAMQP's free Little Lemur tier caps at **20 concurrent connections** (verified 10 Oct 2026), and eleven services plus replicas would exhaust that long before the 1M message quota mattered. `AMQP_URL` remains env-swappable, so CloudAMQP or Amazon MQ is a config change.

Apache Kafka is the planned Phase 3 migration for the scoring event stream specifically, for replay and higher throughput. Because the score event log is already append-only with gap-free sequencing (6.2), this is a producer swap rather than a redesign.

### 7.4 Real-Time Score Flow

```
Organiser quick-tap
  (client generates client_event_id, queues locally, optimistic UI update)
        |
  [fanout-gateway]  (JWT validated, correlation ID injected, rate limited)
        |
  [fanout-scoring-service]
    dedupe on (event_id, client_event_id)        -- idempotent
    validate action against sport config
    BEGIN
      append ScoreEvent, assign seq
      update projection (sport_data, home_score, away_score, events.seq)
    COMMIT
    write Redis hot cache (event:{id}:state, ~2KB)
    publish delta to Redis pub/sub (event:{id}:updates) with seq + prev_seq
        |
  [fanout-websocket-service]
    subscribed to event:{id}:updates
    push delta (~60 bytes) to all connected audience clients
        |
  Audience React clients
    verify prev_seq == last_seq, else refetch full state
    apply delta to Zustand store
    component re-renders
```

Audience clients:
- Fetch full event state once via REST on page open, served from the Redis cache, never from PostgreSQL
- Receive deltas only from the WebSocket thereafter, with gap detection per 6.3
- Fallback: if the WebSocket fails three times, poll `/api/events/{id}/state?since={seq}` every 30 seconds

### 7.5 Mobile Background Behaviour

**Changed from revision 1.** Revision 1 specified dropping the WebSocket on background and switching to FCM silent push for score deltas. This does not work: iOS throttles `content-available` pushes aggressively and unpredictably, and building a correctness-critical path on them guarantees wrong scores on iOS.

Actual behaviour:
- App backgrounded: close the WebSocket, record `last_seq`
- App foregrounded: refetch full state in one call, reopen the socket. The seq mechanism makes this exact rather than approximate.
- FCM is used only for discrete, user-visible notifications: event going live, final result, co-manager invite, followed-organiser activity. Never for score state transport.

Simpler than the original design, and it actually works.

### 7.6 Offline Score Entry (Phase 1)

Promoted from Phase 3, because poor connectivity is the Phase 1 operating condition rather than an edge case.

Organiser taps are written to an IndexedDB queue with a client-generated `client_event_id` and applied optimistically to local state. A background flusher drains the queue in order whenever connectivity allows. Server-side idempotency (6.2) makes replay safe, so the flusher retries aggressively with no risk of double-counting.

The scoring UI shows an explicit pending-sync indicator with a count. An organiser who loses signal mid-over sees their taps land locally and knows they are queued, rather than silently losing them.

This requires no new mechanism - it falls directly out of idempotent score events.

### 7.7 Abuse Controls on the Public Read Path

Share links require no authentication, and opening a WebSocket costs an attacker nothing.

- Cloudflare in front of all traffic: bot fight mode, per-IP rate limiting on `/api/events/**`
- Gateway: per-IP request rate limit, Redis-backed (Resilience4j rate limiter)
- WebSocket Service: max concurrent connections per IP (10), max per event (configurable ceiling), connection rate limit per IP
- Score writes: max 10 per minute per event from the sport config, enforced in Redis before the database is touched
- Public state endpoint responses cached at Cloudflare for 5 seconds - this absorbs the thundering herd when an event link is shared into a large WhatsApp group

### 7.8 Observability

Non-optional with eleven services. Retrofitting distributed tracing is painful, so this goes in with the first service, not the twentieth sprint.

- Correlation ID generated at the gateway, propagated through Feign and RabbitMQ headers, present in every log line
- Structured JSON logging throughout
- Micrometer metrics from every service, scraped by a Prometheus agent
- Grafana Cloud free tier for dashboards and alerting, with metrics and logs shipped off-host so losing the host does not also lose the telemetry
- Spring Boot Actuator health endpoints, consumed by Eureka and by the container restart policy
- Golden signals per service, plus three domain alerts: score write latency p99, WebSocket delta fanout lag, and **gap-resync rate** - a rising resync rate is the early warning that delta delivery is degrading

### 7.9 Local Development

Eleven services on a laptop is the single thing most likely to slow development, so it is designed for rather than discovered.

- `docker-compose.yml` with Compose profiles: `infra` (Postgres, Redis, RabbitMQ), `core` (Eureka, Config, Gateway, Identity, Event, Scoring, WebSocket), `full` (everything)
- Day-to-day work runs `infra` + `core` only - seven containers, roughly 3 GB
- The service under active development runs from the IDE against the composed infra, not in a container
- Testcontainers for integration tests, so there is no shared dev database to corrupt
- Seeded fixtures: one live cricket event, one upcoming football event, one flagged event, one probationary organiser, one admin

### 7.10 CI/CD

- GitHub Actions. One reusable workflow consumed by all eleven services; each service's own config is about twenty lines.
- On PR: compile, unit tests, Testcontainers integration tests, Spotless, OWASP dependency check
- On merge to main: build the container image, push to GitHub Container Registry, deploy only the changed services
- Images are built **multi-arch (arm64 + amd64)**. The host CPU architecture is a deployment choice (section 10), and the build must not be the thing that constrains it.
- Flyway migrations run as a pre-deploy step, per schema, forward-only
- Nightly `pg_dump` of all schemas to object storage, 14-day retention. On a single-node database this is the only thing standing between a disk failure and total loss.

---

## 8. Frontend Architecture (React + Capacitor)

### Web Stack

| Layer | Library |
|---|---|
| Framework | React + Vite |
| Styling | Tailwind CSS |
| Components | shadcn/ui (Radix UI primitives) |
| Animation | Motion (Framer Motion) |
| Icons | Lucide React |
| State (live scores) | Zustand |
| Server state | TanStack Query |
| Offline queue | IndexedDB via idb |
| Forms | React Hook Form + Zod |
| Routing | React Router v7 |
| Charts | Recharts |

### Mobile via Capacitor

Capacitor wraps the React build in a native WebView for Android and iOS.

- `@capacitor/push-notifications` - FCM integration, device token registration
- `@capacitor/geolocation` - location for event discovery
- `@capacitor/haptics` - haptic feedback on scoring button taps
- `@capacitor/network` - connectivity detection driving the offline queue flusher
- `@capacitor/preferences` - durable local storage of the pending score queue
- `@capacitor/splash-screen`, `@capacitor/assets`

Store considerations: thin WebView wrappers do attract review friction. The scoring interface must feel native - haptics on every tap, instant optimistic feedback, no visible web chrome. Geolocation requires a published privacy policy before either store accepts the build.

### Theming

Each event has an organiser-chosen accent colour, applied as a CSS custom property on the event wrapper:

```jsx
<div style={{ "--accent": event.accentColor }}>
  <EventPanel />
</div>
```

Tailwind reads `--accent` as a colour token, so every component inside the wrapper inherits the event colour without prop drilling. Accent colours are validated server-side against a contrast floor, so an organiser's choice cannot produce unreadable text.

Safe area insets are applied globally from app init.

### Project Structure

```
fanout-web/
  src/
    components/
      ui/              -- shadcn components
      scoring/         -- sport-specific quick-tap panels
      event/           -- event card, event page, timeline entry, score panel
      discovery/       -- search bar, event list, location filter
      admin/           -- moderation queue, organiser management
    stores/
      liveScoreStore.ts     -- Zustand store, seq-aware delta application
      pendingScoreQueue.ts  -- IndexedDB-backed offline queue
    hooks/
      useEventSocket.ts     -- socket lifecycle, gap detection, resync
    services/
      api.ts                -- TanStack Query fetchers
    lib/
      sportConfigs/         -- generated from the shared JSON source
    pages/
      Discover.tsx
      EventPage.tsx
      ScoringDashboard.tsx
      AdminPanel.tsx
  android/              -- Capacitor generated, do not edit manually
  ios/                  -- Capacitor generated, do not edit manually
  capacitor.config.ts
```

---

## 9. Moderation Architecture

Four layers, applied in order.

**Layer 1 - Structural gates (zero cost)**
OTP verification required to create events. New organisers serve a probationary period. Event creation requires sport type, both team names, a valid geocoded venue, and date/time. Incomplete submissions are rejected before any content check runs.

**Layer 2 - Rule-based filters (zero cost)**
Local Java profanity and slur library applied to all free-text fields at write time. Geocoding validation on venue address (Nominatim, respecting its usage policy with a contact header and request throttle). Score update rate limiting per sport config. Sham detection: an event marked live with no score update for 45 consecutive minutes is auto-flagged, and the organiser's trust score is debited.

**Layer 3 - Selective AI (minimal cost)**
Called only for event descriptions (once per event), organiser profile and team names (once per registration), and community-reported content.

Score updates are structured data. They never reach a moderation API.

Images use an async hold queue: held pending, moderation runs in the background, published automatically on pass or routed to manual review on fail.

Video reels (Phase 3, paid tiers): one frame every 3–5 seconds through the image pipeline. A 60-second reel becomes 12–20 frames.

Cost at 1,000 events/month: roughly USD 0.08. Google Cloud Vision's first 1,000 SafeSearch units per month are free; beyond that SafeSearch is USD 1.50 per 1,000 units, or free when paired with Label Detection (verified 10 Oct 2026). The provider sits behind a `ContentModerationProvider` interface, so Amazon Rekognition is a swap.

**Layer 4 - Community flagging**
Any authenticated user can flag content in one tap. Weighted flags totalling 3 trigger automatic hiding pending admin review. Flag weight derives from trust score (section 4).

### Regulatory Note

Fanout hosts user-generated content for Indian users and is an intermediary under the IT Rules 2021. Before public launch it needs: a published privacy policy and terms, a named Grievance Officer with contact details on the site, a documented takedown process meeting the statutory response windows, and retention of moderation decisions for the required period. The append-only moderation audit log already satisfies the record-keeping requirement.

This is a launch blocker, not a Phase 3 concern.

---

## 10. Hosting

**Substantially revised.** Revision 1 targeted zero monthly cost and selected providers first. Verification showed the chosen free tiers cannot host the architecture, and that chasing free was the only thing pushing the design toward a monolith. Revision 2 inverts the order: the architecture states its **resource requirement**, and hosting is whichever provider meets it. No provider is load-bearing.

### 10.1 What Revision 1 Got Wrong

Verified 10 October 2026:

| Revision 1 assumption | Verified reality |
|---|---|
| Railway: $5 credit per month | Free plan = **$1/month** credit, 1 vCPU / **0.5 GB RAM**, 1 replica. Free *Trial* = one-time $5, expires in 30 days. Hobby = **$5/month paid** (includes $5 usage). |
| Fly.io: always-on, 3 VMs free | **No free tier for new organisations.** Trial = 2 machine-hours or 7 days. PAYG from USD 2.19/month per 256 MB machine. Egress to India USD 0.12/GB - 6x the North America rate. |
| Supabase: 500 MB, never sleeps | 500 MB correct, but free projects **pause after 1 week of inactivity**, there are **no automatic backups**, no read replicas, 2 active projects max, and 60 direct / 200 pooler connections. |
| Upstash: 10,000 commands/day | **Now 500,000 commands/month** (~16,600/day), 250 MB data, 10 GB bandwidth. Upstash's own FAQ page is stale and still quotes the daily figure. This is *better* than revision 1 assumed and the design sits comfortably inside it. |
| CloudAMQP: 1M messages/month | Correct, but also capped at **20 concurrent connections**, 100 queues, 10,000 queued messages. The connection cap binds long before the message quota. |
| Cloudflare R2: 10 GB, zero egress | Correct. 10 GB-month, 1M Class A ops, 10M Class B ops, free egress, Standard class only. |
| Google Vision: 1,000 units/month free | Correct. |
| Firebase FCM: free at any scale | Correct. |

The conclusion: eleven Spring Boot JVMs cannot run on $1/month and a single 0.5 GB replica. The constraint was never the architecture.

### 10.2 The Requirement

Fanout Phase 1 needs a single always-on host meeting:

| Resource | Minimum | Comfortable |
|---|---|---|
| RAM | 8 GB | 12–16 GB |
| vCPU | 2 | 4 |
| Disk | 60 GB SSD | 100+ GB |
| Architecture | arm64 or amd64 (images are multi-arch) | either |
| Egress | 1 TB/month | 5+ TB/month |
| Uptime | always-on, no scale-to-zero | same |

**Scale-to-zero is disqualifying.** A Spring Boot cold start is 15–40 seconds, and a live score platform that takes half a minute to wake has failed in the exact moment it matters.

Budget: **₹1,000–2,500 per month** for Phase 1 infrastructure, plus the domain (₹800–1,200/year) and, when iOS ships, the Apple Developer account (USD 99/year). Treating hosting as free is what distorted revision 1; a small fixed budget removes the distortion entirely.

### 10.3 Candidate Hosts

Prices as of October 2026; all of these move, so re-check before committing.

| Option | Shape | Approx. cost | Notes |
|---|---|---|---|
| **Indian-region VPS** (DigitalOcean / Linode / Vultr, Mumbai or Bangalore) | 8 GB / 4 vCPU | USD 48/mo (~₹4,000) | Best latency for Indian users. Most expensive. Over budget. |
| **Oracle Cloud PAYG, Ampere A1, `ap-mumbai-1`** | 2 OCPU / 12 GB | ~USD 28/mo (~₹2,300) | India region, no capacity lottery, identical shape to Oracle's free tier - so if free capacity ever appears, moving to it is a billing change, not a migration. |
| **Hetzner CAX line (ARM)** | CAX21 4 vCPU / 8 GB, CAX31 8 vCPU / 16 GB | ~EUR 11–21/mo after the June 2026 price rise | Cheapest per GB. EU regions (Germany/Finland); Singapore CAX availability is **unconfirmed** and the CAX line has intermittently shown as unavailable to order. ~130 ms from India via EU, ~70 ms if Singapore is obtainable. |
| **Oracle Cloud Always Free, Ampere A1** | 2 OCPU / 12 GB | ₹0 | **Opportunistic only.** Oracle halved this allowance in mid-2026 (was 4 OCPU / 24 GB), A1 capacity is frequently unavailable in popular regions, and Oracle has reclaimed idle resources. Not something to plan around. |

**Recommended default: Oracle Cloud PAYG Ampere A1 in `ap-mumbai-1` or `ap-hyderabad-1`.** It is inside budget, it is in-country so latency is genuinely good for the target audience, 12 GB gives comfortable headroom, and it is the *same shape* as Oracle's free tier - meaning a free-capacity windfall is pure upside rather than a different architecture. Hetzner CAX31 is the fallback if cost matters more than Indian latency and the CAX line is orderable at the time.

Everything else stays provider-pinned only where it is genuinely free at scale:

| Component | Platform | Terms |
|---|---|---|
| Object storage + CDN | Cloudflare R2 | 10 GB, 1M Class A, 10M Class B ops, zero egress |
| Web frontend | Cloudflare Pages | Unlimited bandwidth, free |
| DNS, SSL, WAF, ingress | Cloudflare + Cloudflare Tunnel | Free. Tunnel removes the need for a public load balancer or a static IP. |
| Push notifications | Firebase FCM | Free at any scale |
| Image moderation | Google Cloud Vision | 1,000 SafeSearch units/month free |
| Metrics, logs, alerting | Grafana Cloud | Free tier |
| Geocoding | Nominatim | Free; usage policy applies |
| Database backups | `pg_dump` to R2, nightly | Inside R2 free tier |

PostgreSQL, Redis and RabbitMQ run as containers on the application host in Phase 1. This removes three provider ceilings that would otherwise shape the design - Supabase's 60 connections and 1-week pause, Upstash's command quota on a pub/sub-heavy workload, and CloudAMQP's 20-connection cap. `DATABASE_URL`, `REDIS_URL` and `AMQP_URL` all stay environment variables, so moving any one of them to a managed service is a config change (rule 4).

### 10.4 Memory Budget

The architecture is only credible if it demonstrably fits. These are container limits, not aspirations.

| Process | Comfortable (12–16 GB host) | Tight (8 GB host) |
|---|---|---|
| PostgreSQL 16 + PostGIS | 1536 MB | 1024 MB |
| Redis 7 | 256 MB | 192 MB |
| RabbitMQ 3.13 | 512 MB | 384 MB |
| Eureka Server | 320 MB | 256 MB |
| Config Server | 320 MB | 256 MB |
| Gateway | 448 MB | 384 MB |
| Identity Service | 448 MB | 384 MB |
| Event Service | 576 MB | 448 MB |
| Scoring Service | 512 MB | 448 MB |
| WebSocket Service | 512 MB | 448 MB |
| Media Service | 384 MB | 288 MB |
| Moderation Service | 384 MB | 288 MB |
| Notification Service | 384 MB | 288 MB |
| Admin Service | 384 MB | 288 MB |
| Prometheus agent + log shipper | 256 MB | 192 MB |
| **Total** | **~7.2 GB** | **~5.6 GB** |

The tight profile runs all eleven services on an 8 GB host with roughly 2.4 GB left for the OS, page cache and image pulls. No service is merged or dropped to achieve it.

Required JVM settings: `-XX:MaxRAMPercentage=70 -XX:+UseSerialGC -Xss512k`, plus `-XX:TieredStopAtLevel=1` on the four low-traffic services (Eureka, Config, Notification, Admin). Spring Boot 3.x with CDS enabled cuts both startup time and footprint. Build with Paketo buildpacks or Eclipse Temurin base images, multi-arch.

### 10.5 Known Risks and Mitigations

- **Single point of failure.** One host, no managed failover. Mitigation: nightly `pg_dump` to R2 is **not optional**; telemetry ships to Grafana Cloud so a host loss does not also destroy the evidence; infrastructure is defined as a single `docker-compose.yml` plus a provisioning script, so rebuilding on a fresh host is a documented procedure measured in minutes, not an archaeology exercise.
- **Self-managed.** OS patching, container lifecycle and TLS renewal are yours. Cloudflare Tunnel removes TLS termination from the host entirely, which eliminates the most common source of self-managed breakage. Budget the remaining operational time honestly.
- **Provider churn.** Every price and limit in this section was verified on one day and will drift. The mitigation is section 10.2: the requirement is stated in resources, and the deployment target is one provisioning script away from any host that meets it.
- **Capacity availability.** Applies to Oracle A1 shapes in particular, free or paid. Have a second candidate from 10.3 chosen in advance rather than discovering the problem on launch day.

---

## 11. Managed-Service Migration Plan

Migration order is lowest risk first. Every step is a config or environment variable change, because all dependencies sit behind interfaces from day one. The target here is written as AWS, but each step is provider-neutral.

1. **Cloudflare R2 to Amazon S3 + CloudFront** - zero downtime, one-time object copy, swap `STORAGE_ENDPOINT`
2. **Google Vision to Amazon Rekognition** - zero downtime, swap the `ContentModerationProvider` implementation
3. **Self-hosted Redis to ElastiCache / Upstash** - minimal downtime; Redis holds only ephemeral projections and repopulates from the score event log on first cache miss
4. **Self-hosted RabbitMQ to Amazon MQ / CloudAMQP** - brief downtime; all consumers are idempotent and all queues durable, so in-flight messages replay safely
5. **Self-hosted PostgreSQL to RDS Multi-AZ** - approximately 5 minutes planned downtime, with DMS replicating for days beforehand. The Admin Service's read-only view grants become a genuine read replica at this step, with no application change.
6. **Single host to ECS Fargate** - zero downtime via blue-green with weighted DNS shifting (10% / 50% / 100%). Eureka is retired in favour of ECS Service Connect; because all resolution already goes through `lb://service-name`, this is a dependency swap rather than a rewrite.
7. **Cloudflare DNS/CDN** - keep permanently, point the CNAME at the load balancer

AWS Lightsail is explicitly ruled out: it is a VPS product incompatible with independent service scaling, and it adds friction to the eventual Fargate migration.

---

## 12. Monetisation Model

Organiser-side monetisation only. Audience viewing is free at every tier, forever.

**Revised from revision 1.** Revision 1 led with ₹299–499/month subscriptions. That price shape fits the product poorly: most amateur organisers run one or two tournaments a year, so a monthly subscription is paid through eleven idle months and then cancelled. Worse, the free tier's 3-active-event allowance exceeds what most users will ever need - so the free tier satisfies nearly everyone and almost nobody converts.

Per-tournament pricing matches actual usage, and charges at peak motivation: the week of the event.

| Product | Price | Shape | Key features |
|---|---|---|---|
| Free | ₹0 | - | 2 active events, simplified scoring, Fanout branding on shared links, 3-month archive |
| **Tournament Pass** | **₹149–299** | **Per tournament, one-time** | **Unlimited events within one tournament, image uploads, PDF/CSV export, custom branding, clean public URL, no Fanout branding, polls and MVP voting** |
| Club / League | ₹999–1499/month | Subscription | Season management, standings, player profiles, cross-event stats, embed widget, sponsor slot. Usage here is genuinely continuous, so a subscription fits. |
| Venue / Sponsor | Flat fee, B2B | Annual contract | Geographic search placement, analytics dashboard, sold directly |

One-time purchases:
- Verified Organiser badge: ₹199 (identity check, trust signal, +25 trust score)
- Highlight summary cards: ₹49–99 per event

**The Venue/Sponsor tier is probably the real revenue.** A local sports complex or coaching academy has a budget line and a recurring need; a college fest convenor has neither. Build the B2B placement surface earlier than its Phase 3 slot suggests.

Payment processors: Razorpay for India, Stripe for international, both behind a `PaymentProvider` interface.

Phase 1 ships no monetisation. Every price above is a hypothesis to be tested, not a decision - instrument accordingly (section 14).

---

## 13. Non-Negotiable Architectural Rules

These must never be violated regardless of time pressure or convenience.

1. The WebSocket service is always a separate deployed process from the REST API services. They must never share a JVM.
2. Audience clients read live score state from the Redis cache, never from PostgreSQL.
3. Score update submissions never go to an AI moderation API. They are structured data, validated by sport config rules only.
4. All external service credentials and URLs are environment variables. Nothing is hardcoded.
5. Storage operations use the S3-compatible SDK with a configurable endpoint. Changing object storage providers must be a one-line config change.
6. Each microservice owns its own database schema, enforced by per-service database roles - not by convention. Only the Admin Service reads across schemas, and only through views those services explicitly expose.
7. Audience access to event viewing must remain free at every tier, forever.
8. **The score event log is append-only.** Rows are never updated or deleted. Corrections are compensating events. `sport_data` is always a derived projection and must be rebuildable from the log alone.
9. **Every score write is idempotent on a client-generated id.** No exceptions, no "just this one endpoint".
10. **Every delta carries `seq` and `prev_seq`, and every client detects gaps and resyncs.** A client must never be able to silently diverge from server state.
11. **FCM is never a transport for score state.** Notifications only.
12. **No free tier is a design input.** If a limit shapes an architectural decision, that limit is wrong and the host changes - not the architecture.

---

## 14. Product Instrumentation

Phase 2's sport list and the entire pricing model in section 12 are currently guesses. They should be data.

Tracked from Phase 1: events created per sport; events reaching "completed" versus abandoned; score updates per event (the proxy for whether the quick-tap interface actually works); viewers per event and session duration; share-link click-through; sign-up conversion after the 3-minute prompt; offline queue depth and flush latency; gap-resync rate per client platform.

Two questions these answer: which five sports earn Phase 2 slots, and whether anyone would pay for a Tournament Pass. Self-hosted Plausible or Umami on the application host, or PostHog's free tier.

---

## 15. Open Questions

Recorded rather than silently assumed.

1. **OTP delivery cost.** SMS OTP in India runs roughly ₹0.12–0.25 per message with no free tier, and is the only unavoidable per-user cost in the whole stack. At 1,000 organiser registrations plus repeat logins it becomes a real line item. Proposed mitigation: email OTP as the default, SMS as opt-in. Needs a decision before the Identity Service is built.
2. **Who resolves a disputed score?** Two co-managers disagree, or an audience member insists the score is wrong. Last-write-wins settles the *data*; it does not settle the *argument*. Is there an appeal path, and does the audit log become visible to anyone outside admin?
3. **Event deletion versus the append-only log.** Rule 8 says never delete; Indian data protection law gives users deletion rights. Reconcile deliberately - most likely delete personal identifiers and retain the anonymised score log. Decide before launch, not after the first request arrives.
4. **Venue addresses are public**, including school and college grounds, with a live indicator showing when people are gathered there. Worth a deliberate decision about location precision for events at institutions.
5. **Cricket scoring depth versus adoption.** If simplified Phase 1 cricket drives organisers away, the Phase 2 full model becomes a Phase 1 blocker. Instrument sport-level abandonment specifically, to find out early rather than late.
