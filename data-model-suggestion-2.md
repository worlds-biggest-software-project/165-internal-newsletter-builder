# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: Internal Newsletter Builder · Created: 2026-05-20

## Philosophy

This model treats every state change as an immutable event in a central event store. The current state of any entity (subscriber, campaign, template) is derived by replaying its events rather than reading a mutable row. Read-optimised projections (materialised views) serve the application UI and API, while the event store provides a complete, tamper-evident audit trail.

This approach is inspired by event-sourced systems in financial services and healthcare where "what happened and when" is as important as "what is the current state." Email platforms like SendGrid and Mailgun already model delivery as a stream of events (queued, sent, delivered, opened, clicked, bounced). This model extends that philosophy to the entire application: subscriber lifecycle, content authoring, campaign management, and analytics are all event streams.

For an internal newsletter platform where GDPR compliance, consent tracking, and engagement analytics are core requirements, event sourcing provides natural answers: consent history is an immutable log, engagement is a stream of events, and any point-in-time query ("was this employee subscribed on March 15?") is a replay operation.

**Best for:** Organisations that need full audit trails for compliance, temporal queries ("what was the subscriber list when this campaign was sent?"), and AI/ML pipelines that consume event streams for engagement prediction and send-time optimisation.

**Trade-offs:**
- Pro: Complete, immutable audit trail — every change is recorded with timestamp and actor
- Pro: Temporal queries are natural — replay to any point in time
- Pro: Event streams feed directly into ML pipelines for engagement prediction
- Pro: GDPR compliance is built-in — consent changes are an auditable event log
- Pro: Decoupled read/write models allow independent scaling
- Con: Higher complexity — developers must understand event sourcing and CQRS patterns
- Con: Read models must be maintained and rebuilt when projections change
- Con: Event store grows indefinitely — requires snapshotting and archival strategies
- Con: Eventual consistency between event store and read models can confuse users
- Con: Debugging requires tracing through event sequences rather than inspecting rows

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCIM 2.0 (RFC 7643/7644) | SCIM sync operations are recorded as `subscriber.imported`, `subscriber.updated`, `subscriber.deactivated` events with full attribute diffs |
| GDPR Articles 6, 21 | Consent changes are `consent.granted` and `consent.withdrawn` events — the consent history is the event stream itself |
| RFC 8058 (One-Click Unsubscribe) | Unsubscribe actions produce `subscriber.unsubscribed` events with source attribution |
| MJML | Template changes tracked as `template.content_updated` events; any version can be reconstructed by replay |
| OCSF (Open Cybersecurity Schema Framework) | Event schema borrows from OCSF patterns for structured activity logging |
| ISO 3166-1 | Geographic attributes on subscriber events use ISO 3166-1 alpha-2 codes |
| SPF/DKIM/DMARC | Domain verification changes tracked as `domain.verified`, `domain.spf_passed` events |

---

## Event Store (Core)

```sql
-- Central immutable event store
-- This is the single source of truth for the entire system.
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     VARCHAR(50) NOT NULL,
        -- 'subscriber', 'campaign', 'template', 'list', 'organisation',
        -- 'delivery', 'engagement', 'consent', 'ai_job'
    stream_id       UUID NOT NULL,
        -- The aggregate/entity this event belongs to
    event_type      VARCHAR(100) NOT NULL,
        -- e.g. 'subscriber.created', 'campaign.published', 'email.opened'
    event_version   INTEGER NOT NULL,
        -- Monotonically increasing per stream_id for ordering
    payload         JSONB NOT NULL,
        -- The event data — varies by event_type
    metadata        JSONB NOT NULL DEFAULT '{}',
        -- Actor, IP, user-agent, correlation IDs
    organisation_id UUID NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, event_version)
);

-- Partition by month for manageability
-- In production: CREATE TABLE event_store (...) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_type ON event_store(event_type, occurred_at);
CREATE INDEX idx_event_org ON event_store(organisation_id, occurred_at);
CREATE INDEX idx_event_time ON event_store(occurred_at);

-- Snapshots for aggregate reconstruction performance
CREATE TABLE event_snapshot (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    snapshot_version INTEGER NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);
```

### Example Event Payloads

```sql
-- subscriber.created event
-- payload: {
--   "email": "jane.doe@company.com",
--   "first_name": "Jane",
--   "last_name": "Doe",
--   "department": "Engineering",
--   "employee_id": "EMP-1234",
--   "source": "scim",
--   "country_code": "US"
-- }
-- metadata: {
--   "actor_type": "system",
--   "actor_id": "scim-sync-job-789",
--   "correlation_id": "sync-batch-2026-05-20"
-- }

-- campaign.published event
-- payload: {
--   "name": "May Engineering Update",
--   "subject": "What shipped in May",
--   "template_id": "tpl-456",
--   "target_list_ids": ["list-a", "list-b"],
--   "scheduled_at": "2026-05-21T09:00:00Z",
--   "content_source": "ai_generated"
-- }
-- metadata: {
--   "actor_type": "user",
--   "actor_id": "user-321",
--   "ip": "10.0.1.50"
-- }

-- email.opened event
-- payload: {
--   "campaign_id": "camp-789",
--   "subscriber_id": "sub-456",
--   "device_type": "mobile",
--   "email_client": "Apple Mail",
--   "is_proxy": true
-- }

-- consent.withdrawn event
-- payload: {
--   "subscriber_id": "sub-456",
--   "preference_type": "category:team_news",
--   "previous_basis": "consent",
--   "withdrawal_source": "one_click_unsubscribe"
-- }
```

---

## Read Models (CQRS Projections)

These tables are **derived from the event store** and can be rebuilt at any time by replaying events. They serve the application UI and API for fast reads.

```sql
-- ============================================================
-- PROJECTION: Current subscriber state
-- Rebuilt from: subscriber.created, subscriber.updated,
--               subscriber.deactivated, subscriber.reactivated
-- ============================================================
CREATE TABLE read_subscriber (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    department      VARCHAR(255),
    job_title       VARCHAR(255),
    location        VARCHAR(255),
    country_code    CHAR(2),
    timezone        VARCHAR(50),
    employee_id     VARCHAR(100),
    manager_id      UUID,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    source          VARCHAR(50) NOT NULL,
    last_event_version INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE UNIQUE INDEX idx_read_sub_org_email ON read_subscriber(organisation_id, email);
CREATE INDEX idx_read_sub_dept ON read_subscriber(organisation_id, department);
CREATE INDEX idx_read_sub_status ON read_subscriber(organisation_id, status);

-- ============================================================
-- PROJECTION: Current list memberships
-- Rebuilt from: list.created, list.subscriber_added,
--               list.subscriber_removed
-- ============================================================
CREATE TABLE read_list (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    type            VARCHAR(20) NOT NULL,
    subscriber_count INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE TABLE read_list_membership (
    list_id         UUID NOT NULL REFERENCES read_list(id) ON DELETE CASCADE,
    subscriber_id   UUID NOT NULL REFERENCES read_subscriber(id) ON DELETE CASCADE,
    added_at        TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (list_id, subscriber_id)
);

-- ============================================================
-- PROJECTION: Campaign state and content
-- Rebuilt from: campaign.created, campaign.content_updated,
--               campaign.published, campaign.sent, campaign.cancelled
-- ============================================================
CREATE TABLE read_campaign (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    subject         VARCHAR(500) NOT NULL,
    preview_text    VARCHAR(500),
    from_name       VARCHAR(255),
    from_email      VARCHAR(320),
    status          VARCHAR(20) NOT NULL,
    content_html    TEXT,
    content_mjml    TEXT,
    content_source  VARCHAR(50),
    template_id     UUID,
    scheduled_at    TIMESTAMPTZ,
    sent_at         TIMESTAMPTZ,
    total_recipients INTEGER DEFAULT 0,
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_read_camp_org ON read_campaign(organisation_id);
CREATE INDEX idx_read_camp_status ON read_campaign(organisation_id, status);

-- ============================================================
-- PROJECTION: Campaign analytics (aggregated from engagement events)
-- Rebuilt from: email.sent, email.delivered, email.bounced,
--               email.opened, link.clicked
-- ============================================================
CREATE TABLE read_campaign_analytics (
    campaign_id     UUID PRIMARY KEY REFERENCES read_campaign(id),
    total_sent      INTEGER NOT NULL DEFAULT 0,
    total_delivered  INTEGER NOT NULL DEFAULT 0,
    total_bounced   INTEGER NOT NULL DEFAULT 0,
    unique_opens    INTEGER NOT NULL DEFAULT 0,
    total_opens     INTEGER NOT NULL DEFAULT 0,
    unique_clicks   INTEGER NOT NULL DEFAULT 0,
    total_clicks    INTEGER NOT NULL DEFAULT 0,
    open_rate       NUMERIC(5,4),
    click_rate      NUMERIC(5,4),
    bounce_rate     NUMERIC(5,4),
    last_updated    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PROJECTION: Per-subscriber engagement history
-- Rebuilt from: email.opened, link.clicked events
-- ============================================================
CREATE TABLE read_subscriber_engagement (
    subscriber_id   UUID NOT NULL,
    campaign_id     UUID NOT NULL,
    sent_at         TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    first_opened_at TIMESTAMPTZ,
    open_count      INTEGER NOT NULL DEFAULT 0,
    click_count     INTEGER NOT NULL DEFAULT 0,
    device_type     VARCHAR(20),
    email_client    VARCHAR(100),
    is_proxy_open   BOOLEAN DEFAULT false,
    PRIMARY KEY (subscriber_id, campaign_id)
);

CREATE INDEX idx_engagement_campaign ON read_subscriber_engagement(campaign_id);

-- ============================================================
-- PROJECTION: Consent ledger (never deleted, append-only projection)
-- Rebuilt from: consent.granted, consent.withdrawn events
-- ============================================================
CREATE TABLE read_consent_ledger (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id   UUID NOT NULL,
    preference_type VARCHAR(50) NOT NULL,
    action          VARCHAR(20) NOT NULL,
        -- 'granted', 'withdrawn'
    lawful_basis    VARCHAR(50),
    source          VARCHAR(100),
    occurred_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_consent_sub ON read_consent_ledger(subscriber_id, occurred_at);

-- ============================================================
-- PROJECTION: Template version history
-- Rebuilt from: template.created, template.content_updated events
-- ============================================================
CREATE TABLE read_template (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(100),
    current_mjml    TEXT,
    current_html    TEXT,
    version         INTEGER NOT NULL DEFAULT 1,
    created_by      UUID,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
```

## Infrastructure & AI Tables

```sql
-- ============================================================
-- OPERATIONAL: Organisation and user (not event-sourced — low-change config)
-- ============================================================
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255),
    password_hash   VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'editor',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, email)
);

-- ============================================================
-- OPERATIONAL: Sending infrastructure
-- ============================================================
CREATE TABLE sending_domain (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    domain          VARCHAR(255) NOT NULL,
    spf_verified    BOOLEAN NOT NULL DEFAULT false,
    dkim_verified   BOOLEAN NOT NULL DEFAULT false,
    dmarc_verified  BOOLEAN NOT NULL DEFAULT false,
    verified_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, domain)
);

-- ============================================================
-- OPERATIONAL: Projection rebuild tracking
-- ============================================================
CREATE TABLE projection_checkpoint (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID,
    last_event_time TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
        -- 'active', 'rebuilding', 'paused'
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- OPERATIONAL: AI generation tracking (events written to event_store,
-- but this table provides fast lookup for in-progress jobs)
-- ============================================================
CREATE TABLE ai_job_status (
    id              UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    campaign_id     UUID,
    job_type        VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    model_used      VARCHAR(100),
    tokens_used     INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_ai_job_org ON ai_job_status(organisation_id, status);
```

---

## Example Queries

### Temporal Query: "Who was subscribed to list X when campaign Y was sent?"

```sql
-- Step 1: Find when the campaign was sent
SELECT payload->>'sent_at' AS sent_at
FROM event_store
WHERE stream_id = 'campaign-uuid'
  AND event_type = 'campaign.sent'
ORDER BY event_version DESC LIMIT 1;

-- Step 2: Replay list membership events up to that timestamp
SELECT DISTINCT ON (payload->>'subscriber_id')
    payload->>'subscriber_id' AS subscriber_id,
    event_type
FROM event_store
WHERE stream_type = 'list'
  AND stream_id = 'list-uuid'
  AND event_type IN ('list.subscriber_added', 'list.subscriber_removed')
  AND occurred_at <= '2026-05-15T09:00:00Z'  -- campaign sent_at
ORDER BY payload->>'subscriber_id', event_version DESC;
-- Keep only rows where event_type = 'list.subscriber_added'
```

### Consent Audit: "Show full consent history for subscriber Z"

```sql
SELECT event_type, payload, metadata, occurred_at
FROM event_store
WHERE stream_type = 'consent'
  AND payload->>'subscriber_id' = 'sub-uuid'
ORDER BY occurred_at ASC;
```

### Rebuild a Projection

```sql
-- Rebuild read_campaign_analytics from events
TRUNCATE read_campaign_analytics;

INSERT INTO read_campaign_analytics (campaign_id, total_sent, total_delivered,
    total_bounced, unique_opens, total_opens, unique_clicks, total_clicks,
    open_rate, click_rate, bounce_rate, last_updated)
SELECT
    stream_id AS campaign_id,
    COUNT(*) FILTER (WHERE event_type = 'email.sent') AS total_sent,
    COUNT(*) FILTER (WHERE event_type = 'email.delivered') AS total_delivered,
    COUNT(*) FILTER (WHERE event_type = 'email.bounced') AS total_bounced,
    COUNT(DISTINCT payload->>'subscriber_id')
        FILTER (WHERE event_type = 'email.opened') AS unique_opens,
    COUNT(*) FILTER (WHERE event_type = 'email.opened') AS total_opens,
    COUNT(DISTINCT payload->>'subscriber_id')
        FILTER (WHERE event_type = 'link.clicked') AS unique_clicks,
    COUNT(*) FILTER (WHERE event_type = 'link.clicked') AS total_clicks,
    NULL, NULL, NULL,
    now()
FROM event_store
WHERE stream_type = 'delivery'
  AND event_type IN ('email.sent','email.delivered','email.bounced',
                     'email.opened','link.clicked')
GROUP BY stream_id;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 2 | event_store, event_snapshot |
| Read Models — Subscribers | 2 | read_subscriber, read_consent_ledger |
| Read Models — Lists | 2 | read_list, read_list_membership |
| Read Models — Campaigns | 2 | read_campaign, read_campaign_analytics |
| Read Models — Engagement | 1 | read_subscriber_engagement |
| Read Models — Templates | 1 | read_template |
| Operational | 5 | organisation, app_user, sending_domain, projection_checkpoint, ai_job_status |
| **Total** | **15** | Plus the event_store which contains all domain data |

---

## Key Design Decisions

1. **Single event store table** — all domain events flow into one partitioned table rather than per-aggregate stores. This simplifies cross-aggregate queries (e.g., "all events for organisation X") and enables a single event bus consumer.

2. **JSONB payloads** — event data varies by type, so JSONB is the natural fit. The `event_type` field provides schema-level typing; application code validates payload shape per event type.

3. **Read models are disposable** — every `read_*` table can be dropped and rebuilt from the event store. This means schema changes to projections require no data migration — just rebuild.

4. **Engagement events in the event store** — opens, clicks, bounces, and deliveries are events like any other. This unifies analytics with the audit trail and enables ML pipelines to consume a single event stream.

5. **Consent as an event stream** — rather than a mutable flag, consent is a sequence of `consent.granted` and `consent.withdrawn` events. The current state is the latest event; the full history is the stream. This directly satisfies GDPR Article 7(1) requirement to demonstrate consent.

6. **Snapshots for performance** — `event_snapshot` stores periodic aggregate state to avoid replaying thousands of events for entities with long histories. Snapshots are created every N events.

7. **Projection checkpoints** — `projection_checkpoint` tracks which event each read model has processed, enabling incremental updates and crash recovery without replaying the entire store.

8. **Operational tables for low-change config** — organisation, user, and sending domain tables are traditional mutable rows since they change rarely and don't benefit from event sourcing overhead.

9. **Partitioning strategy** — the event store should be partitioned by `occurred_at` (monthly or quarterly) to manage table size. Old partitions can be archived to cold storage while remaining queryable.

10. **Event version for ordering** — `event_version` is monotonically increasing per `stream_id`, providing a total order within each aggregate. Combined with `occurred_at`, this enables both causal and temporal ordering.
