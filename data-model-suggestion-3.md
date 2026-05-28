# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Internal Newsletter Builder · Created: 2026-05-20

## Philosophy

This model uses PostgreSQL's relational strengths for core entities (subscribers, campaigns, lists) while leveraging JSONB columns for areas where flexibility matters most: template content block definitions, personalisation rules, subscriber attributes from diverse HRIS systems, engagement event metadata, and AI generation parameters.

The philosophy is pragmatic: a newsletter platform must handle a wide variety of template structures, personalisation token schemas, and HRIS-sourced employee attributes that vary by customer. Rather than creating dozens of narrow tables or forcing a full event-sourced architecture, JSONB columns absorb this variability while relational foreign keys maintain referential integrity for the core domain graph.

This is the architecture used by modern SaaS platforms that need to ship fast while supporting enterprise-grade features. Products like Notion, Linear, and Posthog use this hybrid pattern — strong relational structure for the domain model, JSONB for user-defined configuration and variable-shape data.

**Best for:** Teams that want to move fast with an MVP, support diverse customer configurations without schema migrations, and avoid the operational complexity of event sourcing while retaining strong relational integrity for core entities.

**Trade-offs:**
- Pro: Rapid development — new subscriber attributes or template block types require no schema migration
- Pro: Single database technology (PostgreSQL) — no need for separate document stores or event databases
- Pro: JSONB columns are indexable with GIN indexes for fast containment and existence queries
- Pro: Balances flexibility with integrity — core relationships are FK-enforced, variable data is JSONB
- Pro: Lower table count (~18 tables) than fully normalized model
- Con: JSONB columns can become "schema-free black boxes" without application-level validation
- Con: Complex JSONB queries can be harder to optimise than simple column queries
- Con: No built-in audit trail for changes within JSONB fields (need application-level tracking)
- Con: Reporting on JSONB fields requires familiarity with PostgreSQL JSONB operators
- Con: JSONB field evolution must be managed in application code rather than database migrations

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCIM 2.0 (RFC 7643/7644) | Core SCIM attributes (email, name, department) are relational columns; extended/custom SCIM attributes stored in `subscriber.custom_attributes` JSONB |
| MJML | Template definitions stored as JSONB content block arrays in `template.blocks`; MJML source per block rather than a single monolithic template |
| JSON Schema | `template.block_schema` references a JSON Schema definition that validates the structure of content blocks |
| RFC 8058 (One-Click Unsubscribe) | `subscriber.preferences` JSONB tracks opt-in/opt-out per category with timestamps |
| WCAG 2.2 | Accessibility check results stored in `campaign.accessibility_report` JSONB |
| GDPR Articles 6, 21 | Consent records embedded in `subscriber.consent_history` JSONB array with timestamps and lawful basis |
| SPF/DKIM/DMARC | DNS verification status in `sending_domain.dns_config` JSONB |
| ISO 3166-1 | `subscriber.country_code` relational column using ISO 3166-1 alpha-2 |
| OpenAPI 3.1 | API responses return JSONB fields directly, matching the flexible schema expectations of modern API consumers |

---

## Organisation & Users

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "default_from_name": "Company Updates",
        --   "default_from_email": "news@company.com",
        --   "branding": {"primary_color": "#1a73e8", "logo_url": "..."},
        --   "ai_enabled": true,
        --   "default_timezone": "America/New_York",
        --   "delivery_provider": "ses"
        -- }
    integrations    JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "hris": {"provider": "workday", "scim_endpoint": "...", "last_synced_at": "..."},
        --   "sso": {"provider": "azure_ad", "tenant_id": "..."},
        --   "slack": {"webhook_url": "...", "channels": ["#comms-team"]},
        --   "delivery": {"provider": "ses", "region": "us-east-1", "config_set": "newsletters"}
        -- }
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
    preferences     JSONB NOT NULL DEFAULT '{}',
        -- {"theme": "dark", "notifications": {"email": true, "slack": false}}
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, email)
);

CREATE INDEX idx_app_user_org ON app_user(organisation_id);

-- Sending domains with flexible DNS config
CREATE TABLE sending_domain (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    domain          VARCHAR(255) NOT NULL,
    dns_config      JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "spf": {"verified": true, "record": "v=spf1 include:amazonses.com ~all"},
        --   "dkim": {"verified": true, "selector": "ses1", "public_key": "..."},
        --   "dmarc": {"verified": true, "policy": "reject"},
        --   "verification_token": "abc123",
        --   "verified_at": "2026-05-01T00:00:00Z"
        -- }
    is_verified     BOOLEAN GENERATED ALWAYS AS (
        (dns_config->'spf'->>'verified')::boolean IS TRUE AND
        (dns_config->'dkim'->>'verified')::boolean IS TRUE
    ) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, domain)
);
```

## Subscribers & Lists

```sql
-- Subscribers with flexible attributes
CREATE TABLE subscriber (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    department      VARCHAR(255),
    job_title       VARCHAR(255),
    country_code    CHAR(2),
    timezone        VARCHAR(50),
    employee_id     VARCHAR(100),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    source          VARCHAR(50) NOT NULL DEFAULT 'manual',

    -- Flexible attributes from HRIS (varies by provider)
    custom_attributes JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "cost_center": "CC-4500",
        --   "division": "Product & Engineering",
        --   "office": "SF-HQ-4th-Floor",
        --   "start_date": "2024-03-15",
        --   "manager_email": "boss@company.com",
        --   "scim_external_id": "workday-12345",
        --   "employment_type": "full_time",
        --   "preferred_language": "en"
        -- }

    -- Email preferences and consent (GDPR-ready)
    preferences     JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "categories": {
        --     "company_updates": {"opted_in": true, "basis": "legitimate_interests"},
        --     "team_news": {"opted_in": true, "basis": "consent", "consented_at": "2026-01-15"},
        --     "social_events": {"opted_in": false, "withdrawn_at": "2026-03-01"}
        --   },
        --   "frequency": "weekly",
        --   "format": "html"
        -- }

    -- Engagement profile (ML-derived, updated periodically)
    engagement_profile JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "engagement_score": 0.82,
        --   "segment": "highly_engaged",
        --   "avg_open_rate": 0.75,
        --   "avg_click_rate": 0.32,
        --   "best_send_hour_utc": 14,
        --   "best_send_day": 2,
        --   "preferred_device": "mobile",
        --   "preferred_client": "Apple Mail",
        --   "last_opened_at": "2026-05-18T14:22:00Z",
        --   "topics_of_interest": ["product_updates", "team_wins"],
        --   "churn_risk": 0.12
        -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, email)
);

CREATE INDEX idx_subscriber_org ON subscriber(organisation_id);
CREATE INDEX idx_subscriber_status ON subscriber(organisation_id, status);
CREATE INDEX idx_subscriber_dept ON subscriber(organisation_id, department);
CREATE INDEX idx_subscriber_custom ON subscriber USING GIN (custom_attributes);
CREATE INDEX idx_subscriber_engagement ON subscriber USING GIN (engagement_profile);

-- Mailing lists with flexible filter definitions
CREATE TABLE list (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    type            VARCHAR(20) NOT NULL DEFAULT 'manual',

    -- Dynamic list filter criteria (evaluated at send time)
    filter_rules    JSONB,
        -- {
        --   "operator": "AND",
        --   "conditions": [
        --     {"field": "department", "op": "in", "values": ["Engineering", "Product"]},
        --     {"field": "country_code", "op": "eq", "value": "US"},
        --     {"field": "custom_attributes.division", "op": "eq", "value": "Product & Engineering"},
        --     {"field": "engagement_profile.engagement_score", "op": "gte", "value": 0.5}
        --   ]
        -- }

    subscriber_count INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_list_org ON list(organisation_id);

-- Junction: subscribers <-> manual lists
CREATE TABLE subscriber_list (
    subscriber_id   UUID NOT NULL REFERENCES subscriber(id) ON DELETE CASCADE,
    list_id         UUID NOT NULL REFERENCES list(id) ON DELETE CASCADE,
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (subscriber_id, list_id)
);

CREATE INDEX idx_sub_list_list ON subscriber_list(list_id);
```

## Templates & Content

```sql
-- Templates stored as JSONB block arrays
CREATE TABLE template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    category        VARCHAR(100),
    version         INTEGER NOT NULL DEFAULT 1,

    -- Template structure as an array of typed content blocks
    blocks          JSONB NOT NULL DEFAULT '[]',
        -- [
        --   {
        --     "id": "block-1",
        --     "type": "header",
        --     "mjml": "<mj-section><mj-column><mj-image src=\"{{logo_url}}\" /></mj-column></mj-section>",
        --     "personalisation_tokens": ["logo_url"]
        --   },
        --   {
        --     "id": "block-2",
        --     "type": "text",
        --     "mjml": "<mj-section><mj-column><mj-text>{{greeting}}, here is your update.</mj-text></mj-column></mj-section>",
        --     "personalisation_tokens": ["greeting"],
        --     "ai_editable": true
        --   },
        --   {
        --     "id": "block-3",
        --     "type": "image",
        --     "mjml": "<mj-section><mj-column><mj-image src=\"{{hero_image}}\" alt=\"{{hero_alt}}\" /></mj-column></mj-section>",
        --     "personalisation_tokens": ["hero_image", "hero_alt"],
        --     "wcag_requires_alt": true
        --   },
        --   {
        --     "id": "block-4",
        --     "type": "button",
        --     "mjml": "<mj-section><mj-column><mj-button href=\"{{cta_url}}\">{{cta_text}}</mj-button></mj-column></mj-section>",
        --     "personalisation_tokens": ["cta_url", "cta_text"],
        --     "tracked": true
        --   }
        -- ]

    -- Global template settings
    settings        JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "font_family": "Arial, sans-serif",
        --   "background_color": "#ffffff",
        --   "text_color": "#333333",
        --   "link_color": "#1a73e8",
        --   "max_width": "600px"
        -- }

    compiled_html   TEXT,           -- cached compiled output
    thumbnail_url   VARCHAR(500),
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_template_org ON template(organisation_id);

-- Media library
CREATE TABLE media (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    filename        VARCHAR(255) NOT NULL,
    content_type    VARCHAR(100) NOT NULL,
    file_size       BIGINT NOT NULL,
    storage_path    VARCHAR(500) NOT NULL,
    alt_text        VARCHAR(500),
    dimensions      JSONB,
        -- {"width": 1200, "height": 630}
    uploaded_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_media_org ON media(organisation_id);
```

## Campaigns & Delivery

```sql
-- Campaigns with JSONB content and personalisation
CREATE TABLE campaign (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    subject         VARCHAR(500) NOT NULL,
    preview_text    VARCHAR(500),
    from_name       VARCHAR(255),
    from_email      VARCHAR(320),
    reply_to        VARCHAR(320),
    template_id     UUID REFERENCES template(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',

    -- Content blocks (copied from template, then customised per campaign)
    content_blocks  JSONB NOT NULL DEFAULT '[]',
        -- Same structure as template.blocks, but with actual content filled in

    -- Personalisation rules
    personalisation JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "variants": [
        --     {
        --       "name": "engineering_variant",
        --       "conditions": {"department": "Engineering"},
        --       "block_overrides": {
        --         "block-2": {"mjml": "<mj-text>Engineering team highlights...</mj-text>"}
        --       }
        --     },
        --     {
        --       "name": "sales_variant",
        --       "conditions": {"department": "Sales"},
        --       "block_overrides": {
        --         "block-2": {"mjml": "<mj-text>Sales team highlights...</mj-text>"}
        --       }
        --     }
        --   ],
        --   "default_tokens": {
        --     "greeting": "Hi {{first_name}}",
        --     "logo_url": "https://cdn.example.com/logo.png"
        --   }
        -- }

    -- Target audience
    target_lists    UUID[] NOT NULL DEFAULT '{}',
        -- Array of list IDs to send to

    -- AI generation metadata
    ai_metadata     JSONB,
        -- {
        --   "generated": true,
        --   "model": "claude-opus-4-6",
        --   "input_sources": [
        --     {"type": "slack", "channel": "#eng-updates", "date_range": "2026-05-13/2026-05-20"},
        --     {"type": "meeting_notes", "doc_id": "doc-123"}
        --   ],
        --   "subject_suggestions": [
        --     {"text": "What shipped this week", "predicted_open_rate": 0.68},
        --     {"text": "May Engineering Highlights", "predicted_open_rate": 0.62}
        --   ],
        --   "readability_score": 72,
        --   "tokens_used": 3450,
        --   "generated_at": "2026-05-20T10:00:00Z"
        -- }

    -- Accessibility report (WCAG 2.2)
    accessibility_report JSONB,
        -- {
        --   "passed": false,
        --   "checks": [
        --     {"type": "alt_text", "severity": "error", "count": 2, "message": "2 images missing alt text"},
        --     {"type": "color_contrast", "severity": "warning", "count": 1, "message": "Low contrast on block-4"}
        --   ],
        --   "checked_at": "2026-05-20T10:05:00Z"
        -- }

    scheduled_at    TIMESTAMPTZ,
    sent_at         TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaign_org ON campaign(organisation_id);
CREATE INDEX idx_campaign_status ON campaign(organisation_id, status);
CREATE INDEX idx_campaign_scheduled ON campaign(scheduled_at) WHERE status = 'scheduled';
CREATE INDEX idx_campaign_ai ON campaign USING GIN (ai_metadata) WHERE ai_metadata IS NOT NULL;
```

## Analytics & Engagement Events

```sql
-- Delivery and engagement events (append-only, high volume)
CREATE TABLE engagement_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    campaign_id     UUID NOT NULL REFERENCES campaign(id) ON DELETE CASCADE,
    subscriber_id   UUID NOT NULL REFERENCES subscriber(id) ON DELETE CASCADE,
    event_type      VARCHAR(30) NOT NULL,
        -- 'sent', 'delivered', 'bounced', 'opened', 'clicked', 'unsubscribed'
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Event-specific data in JSONB (varies by event_type)
    event_data      JSONB NOT NULL DEFAULT '{}',
        -- For 'sent': {"provider_message_id": "ses-abc123", "variant": "engineering_variant"}
        -- For 'bounced': {"bounce_type": "hard", "reason": "mailbox not found"}
        -- For 'opened': {"device_type": "mobile", "email_client": "Apple Mail", "is_proxy": true, "ip": "1.2.3.4"}
        -- For 'clicked': {"link_url": "https://...", "link_position": 3, "device_type": "desktop"}
        -- For 'unsubscribed': {"source": "one_click", "category": "team_news"}
);

-- Partition by month for manageability at scale
-- CREATE TABLE engagement_event (...) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_engage_campaign ON engagement_event(campaign_id, event_type);
CREATE INDEX idx_engage_subscriber ON engagement_event(subscriber_id, occurred_at);
CREATE INDEX idx_engage_time ON engagement_event(occurred_at);
CREATE INDEX idx_engage_type ON engagement_event(event_type, occurred_at);
CREATE INDEX idx_engage_data ON engagement_event USING GIN (event_data);

-- Aggregated campaign analytics (materialised for dashboard performance)
CREATE TABLE campaign_analytics (
    campaign_id     UUID PRIMARY KEY REFERENCES campaign(id) ON DELETE CASCADE,
    total_recipients INTEGER NOT NULL DEFAULT 0,
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

    -- Heatmap and breakdown data
    breakdown       JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "by_department": {
        --     "Engineering": {"sent": 120, "opens": 95, "clicks": 40},
        --     "Sales": {"sent": 80, "opens": 55, "clicks": 22}
        --   },
        --   "by_device": {"mobile": 60, "desktop": 85, "tablet": 5},
        --   "by_client": {"Apple Mail": 70, "Gmail": 50, "Outlook": 30},
        --   "by_hour": {"9": 45, "10": 30, "11": 20, "14": 35},
        --   "link_clicks": [
        --     {"url": "https://...", "position": 1, "clicks": 42},
        --     {"url": "https://...", "position": 2, "clicks": 18}
        --   ]
        -- }

    calculated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Queries

### Dynamic List Evaluation with JSONB Filters

```sql
-- Evaluate a dynamic list filter at send time
-- Filter: department IN ('Engineering', 'Product') AND engagement_score >= 0.5
SELECT s.*
FROM subscriber s
WHERE s.organisation_id = 'org-uuid'
  AND s.status = 'active'
  AND s.department IN ('Engineering', 'Product')
  AND (s.engagement_profile->>'engagement_score')::numeric >= 0.5;
```

### Query Custom HRIS Attributes

```sql
-- Find all subscribers in cost center CC-4500
SELECT id, email, first_name, last_name
FROM subscriber
WHERE organisation_id = 'org-uuid'
  AND custom_attributes @> '{"cost_center": "CC-4500"}';

-- Find all subscribers whose preferred language is Spanish
SELECT id, email
FROM subscriber
WHERE organisation_id = 'org-uuid'
  AND custom_attributes @> '{"preferred_language": "es"}';
```

### Engagement Analytics by Department

```sql
-- Get campaign open rate by department from the pre-computed breakdown
SELECT
    campaign_id,
    dept.key AS department,
    (dept.value->>'sent')::int AS sent,
    (dept.value->>'opens')::int AS opens,
    ROUND((dept.value->>'opens')::numeric / NULLIF((dept.value->>'sent')::numeric, 0), 4) AS open_rate
FROM campaign_analytics,
     jsonb_each(breakdown->'by_department') AS dept(key, value)
WHERE campaign_id = 'campaign-uuid';
```

### Find AI-Generated Campaigns

```sql
SELECT id, name, subject, ai_metadata->>'model' AS model,
       ai_metadata->>'readability_score' AS readability
FROM campaign
WHERE organisation_id = 'org-uuid'
  AND ai_metadata IS NOT NULL
  AND ai_metadata @> '{"generated": true}'
ORDER BY created_at DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Users | 3 | organisation, app_user, sending_domain |
| Subscribers & Lists | 3 | subscriber, list, subscriber_list |
| Templates & Content | 2 | template, media |
| Campaigns | 1 | campaign (with JSONB for content, personalisation, AI metadata, accessibility) |
| Analytics | 2 | engagement_event, campaign_analytics |
| **Total** | **11** | JSONB columns absorb complexity that would otherwise require additional tables |

---

## Key Design Decisions

1. **JSONB for subscriber attributes** — `custom_attributes` absorbs the diversity of HRIS data across Workday, BambooHR, SAP SuccessFactors, etc. Core fields (email, department, country_code) remain relational for fast filtering; provider-specific fields go into JSONB without schema migrations.

2. **Engagement profile on the subscriber** — `engagement_profile` JSONB stores ML-derived metrics (engagement score, best send time, churn risk) directly on the subscriber row. This avoids a separate table and a JOIN for every personalisation decision. Updated periodically by background ML jobs.

3. **Template blocks as JSONB arrays** — rather than a `template_block` table with foreign keys, blocks are stored as an ordered JSONB array. This preserves block ordering naturally, enables drag-and-drop reordering without UPDATE cascades, and stores MJML per block for fine-grained AI editing.

4. **Campaign content blocks copied from template** — `campaign.content_blocks` is a snapshot of the template blocks at campaign creation time, then customised. This decouples campaigns from template changes (editing a template does not alter sent campaigns).

5. **Personalisation variants in JSONB** — `campaign.personalisation` defines audience-specific content overrides as a JSONB structure. The application renders different email versions by matching subscriber attributes against variant conditions and applying block overrides.

6. **Single `engagement_event` table** — all delivery and engagement events (sent, delivered, bounced, opened, clicked, unsubscribed) go into one append-only table with event-type-specific data in `event_data` JSONB. This is simpler than separate tables per event type and supports partitioning by time.

7. **GIN indexes on JSONB** — `custom_attributes`, `engagement_profile`, `event_data`, and `ai_metadata` all have GIN indexes to support containment queries (`@>`) and existence checks (`?`). This makes JSONB queries performant for common access patterns.

8. **`target_lists` as UUID array** — using PostgreSQL's native array type for the campaign-to-list relationship avoids a junction table for this simple many-to-many. The `ANY()` operator provides efficient querying.

9. **AI metadata on the campaign** — rather than a separate `ai_generation_job` table, AI provenance (model, input sources, token usage, subject line suggestions) is stored as JSONB on the campaign itself. This keeps the AI context co-located with the content it produced.

10. **Analytics breakdown in JSONB** — `campaign_analytics.breakdown` stores pre-computed department, device, client, and hourly breakdowns as a single JSONB document rather than separate narrow tables. This makes the dashboard API a single row fetch per campaign.

11. **Generated column for domain verification** — `sending_domain.is_verified` is a PostgreSQL GENERATED ALWAYS AS stored column derived from the JSONB dns_config, providing a fast boolean filter without parsing JSONB at query time.
