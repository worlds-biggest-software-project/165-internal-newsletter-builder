# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Internal Newsletter Builder · Created: 2026-05-20

## Philosophy

This model follows classical relational database design with full normalization (3NF+). Every concept in the domain — subscribers, lists, campaigns, templates, content blocks, delivery events, analytics — receives its own dedicated table with strict foreign key relationships. The design prioritises data integrity, query flexibility, and standards alignment over development speed.

This approach mirrors how mature email platforms like Mailchimp structure their backend: audiences (lists) contain members (subscribers), campaigns reference templates, and delivery/engagement events are tracked in separate normalized tables. It is the most predictable architecture for teams with strong SQL expertise and complex reporting requirements.

**Best for:** Teams that need maximum query flexibility, complex cross-entity reporting (e.g., "which departments had the highest engagement with AI-generated content in Q3?"), and strict referential integrity for compliance auditing.

**Trade-offs:**
- Pro: Maximum query flexibility — any ad-hoc report can be written with standard SQL joins
- Pro: Referential integrity enforced at the database level prevents orphaned records
- Pro: Well-understood by most developers; extensive tooling support (ORMs, migration tools)
- Pro: Clear upgrade path — adding features means adding tables, not altering JSONB structures
- Con: Higher table count (~35-40 tables) increases migration complexity
- Con: Many-to-many junction tables add write overhead for common operations
- Con: Schema changes require migrations; less flexible for rapid prototyping
- Con: Complex queries with many joins can be slower than denormalized alternatives

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| SCIM 2.0 (RFC 7643/7644) | `subscriber` table fields map directly to SCIM User attributes (userName, emails, name, active, department, manager) for HRIS sync |
| RFC 5322 (Internet Message Format) | `campaign_send` tracks From, Reply-To, Subject headers per RFC 5322 structure |
| RFC 8058 (One-Click Unsubscribe) | `subscriber_preference` table stores List-Unsubscribe-Post compliance data |
| MJML | `template` table stores MJML source; compiled HTML cached in `template.compiled_html` |
| WCAG 2.2 | `accessibility_check` table records per-campaign accessibility audit results |
| GDPR Articles 6, 21 | `consent_record` table tracks lawful basis and objection rights per subscriber |
| SPF/DKIM/DMARC (RFCs 7208, 6376, 7489) | `sending_domain` table stores DNS authentication configuration per domain |
| OAuth 2.0 (RFC 6749) | `oauth_connection` table stores tokens for HRIS and identity provider integrations |
| ISO 3166-1 | `subscriber.country_code` uses ISO 3166-1 alpha-2 codes for geographic segmentation |

---

## Organisation & Identity

```sql
-- Multi-tenant organisation
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organisation_slug ON organisation(slug);

-- Users who author/manage newsletters
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255),
    password_hash   VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'editor',
        -- roles: owner, admin, editor, viewer
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, email)
);

CREATE INDEX idx_app_user_org ON app_user(organisation_id);
CREATE INDEX idx_app_user_email ON app_user(email);

-- OAuth/SSO connections for HRIS and identity providers
CREATE TABLE oauth_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    provider        VARCHAR(50) NOT NULL,
        -- e.g. 'azure_ad', 'okta', 'google_workspace', 'workday'
    provider_tenant VARCHAR(255),
    access_token    TEXT,
    refresh_token   TEXT,
    token_expires_at TIMESTAMPTZ,
    scim_endpoint   VARCHAR(500),
    sync_status     VARCHAR(50) DEFAULT 'pending',
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_oauth_org ON oauth_connection(organisation_id);
```

## Sending Infrastructure

```sql
-- Sending domains with DNS authentication config
CREATE TABLE sending_domain (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    domain          VARCHAR(255) NOT NULL,
    spf_verified    BOOLEAN NOT NULL DEFAULT false,
    dkim_verified   BOOLEAN NOT NULL DEFAULT false,
    dmarc_verified  BOOLEAN NOT NULL DEFAULT false,
    dkim_selector   VARCHAR(100),
    dkim_private_key TEXT,
    verification_token VARCHAR(255),
    verified_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, domain)
);

-- Email delivery provider configuration
CREATE TABLE delivery_provider (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    provider_type   VARCHAR(50) NOT NULL,
        -- 'smtp', 'sendgrid', 'ses', 'mailgun'
    config          JSONB NOT NULL DEFAULT '{}',
        -- e.g. {"host": "smtp.example.com", "port": 587, "username": "..."}
    is_default      BOOLEAN NOT NULL DEFAULT false,
    sending_domain_id UUID REFERENCES sending_domain(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Subscribers & Lists

```sql
-- Newsletter subscribers (employees)
CREATE TABLE subscriber (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    department      VARCHAR(255),
    job_title       VARCHAR(255),
    location        VARCHAR(255),
    country_code    CHAR(2),           -- ISO 3166-1 alpha-2
    timezone        VARCHAR(50),       -- IANA timezone
    employee_id     VARCHAR(100),      -- from HRIS via SCIM
    manager_id      UUID REFERENCES subscriber(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
        -- 'active', 'unsubscribed', 'bounced', 'complained'
    scim_external_id VARCHAR(255),     -- SCIM externalId for sync
    source          VARCHAR(50) NOT NULL DEFAULT 'manual',
        -- 'manual', 'scim', 'csv_import', 'api'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organisation_id, email)
);

CREATE INDEX idx_subscriber_org ON subscriber(organisation_id);
CREATE INDEX idx_subscriber_email ON subscriber(email);
CREATE INDEX idx_subscriber_dept ON subscriber(organisation_id, department);
CREATE INDEX idx_subscriber_status ON subscriber(organisation_id, status);
CREATE INDEX idx_subscriber_scim ON subscriber(organisation_id, scim_external_id);

-- Mailing lists / audience segments
CREATE TABLE list (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    type            VARCHAR(20) NOT NULL DEFAULT 'manual',
        -- 'manual', 'dynamic', 'all_employees'
    dynamic_filter  JSONB,
        -- For dynamic lists: {"department": ["Engineering", "Product"], "location": "US"}
    subscriber_count INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_list_org ON list(organisation_id);

-- Junction: subscribers <-> lists
CREATE TABLE subscriber_list (
    subscriber_id   UUID NOT NULL REFERENCES subscriber(id) ON DELETE CASCADE,
    list_id         UUID NOT NULL REFERENCES list(id) ON DELETE CASCADE,
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    added_by        UUID REFERENCES app_user(id),
    PRIMARY KEY (subscriber_id, list_id)
);

CREATE INDEX idx_sub_list_list ON subscriber_list(list_id);

-- Subscriber email preferences and consent tracking
CREATE TABLE subscriber_preference (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id   UUID NOT NULL REFERENCES subscriber(id) ON DELETE CASCADE,
    preference_type VARCHAR(50) NOT NULL,
        -- 'all_newsletters', 'category:company_updates', 'category:team_news'
    opted_in        BOOLEAN NOT NULL DEFAULT true,
    lawful_basis    VARCHAR(50),
        -- GDPR: 'legitimate_interests', 'consent'
    consent_recorded_at TIMESTAMPTZ,
    consent_source  VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(subscriber_id, preference_type)
);

CREATE INDEX idx_sub_pref_sub ON subscriber_preference(subscriber_id);
```

## Templates & Content

```sql
-- Email templates (MJML source)
CREATE TABLE template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    category        VARCHAR(100),
        -- 'company_update', 'team_newsletter', 'executive_memo', 'event'
    mjml_source     TEXT NOT NULL,
    compiled_html   TEXT,
    thumbnail_url   VARCHAR(500),
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_template_org ON template(organisation_id);

-- Media / asset library
CREATE TABLE media (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    filename        VARCHAR(255) NOT NULL,
    content_type    VARCHAR(100) NOT NULL,
    file_size       BIGINT NOT NULL,
    storage_path    VARCHAR(500) NOT NULL,
    alt_text        VARCHAR(500),   -- WCAG 2.2 accessibility
    width           INTEGER,
    height          INTEGER,
    uploaded_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_media_org ON media(organisation_id);
```

## Campaigns & Delivery

```sql
-- Newsletter campaigns
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
        -- 'draft', 'scheduled', 'sending', 'sent', 'paused', 'cancelled'
    content_source  VARCHAR(50),
        -- 'manual', 'ai_generated', 'hybrid'
    scheduled_at    TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    sending_domain_id UUID REFERENCES sending_domain(id),
    delivery_provider_id UUID REFERENCES delivery_provider(id),
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_campaign_org ON campaign(organisation_id);
CREATE INDEX idx_campaign_status ON campaign(organisation_id, status);
CREATE INDEX idx_campaign_scheduled ON campaign(scheduled_at) WHERE status = 'scheduled';

-- Content blocks within a campaign
CREATE TABLE campaign_content_block (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES campaign(id) ON DELETE CASCADE,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    block_type      VARCHAR(50) NOT NULL,
        -- 'heading', 'text', 'image', 'button', 'divider', 'social', 'video'
    heading         VARCHAR(500),
    body_html       TEXT,
    body_mjml       TEXT,
    image_media_id  UUID REFERENCES media(id),
    link_url        VARCHAR(1000),
    is_personalised BOOLEAN NOT NULL DEFAULT false,
    ai_generated    BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_content_block_campaign ON campaign_content_block(campaign_id);

-- Junction: campaigns <-> target lists
CREATE TABLE campaign_list (
    campaign_id     UUID NOT NULL REFERENCES campaign(id) ON DELETE CASCADE,
    list_id         UUID NOT NULL REFERENCES list(id) ON DELETE CASCADE,
    PRIMARY KEY (campaign_id, list_id)
);

-- Individual email sends (one row per recipient per campaign)
CREATE TABLE campaign_send (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES campaign(id) ON DELETE CASCADE,
    subscriber_id   UUID NOT NULL REFERENCES subscriber(id) ON DELETE CASCADE,
    status          VARCHAR(20) NOT NULL DEFAULT 'queued',
        -- 'queued', 'sent', 'delivered', 'bounced', 'failed'
    personalisation_variant VARCHAR(100),
    sent_at         TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    bounce_type     VARCHAR(20),
        -- 'hard', 'soft'
    bounce_reason   TEXT,
    provider_message_id VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_send_campaign ON campaign_send(campaign_id);
CREATE INDEX idx_send_subscriber ON campaign_send(subscriber_id);
CREATE INDEX idx_send_status ON campaign_send(campaign_id, status);
```

## Analytics & Engagement

```sql
-- Tracked links in campaigns
CREATE TABLE tracked_link (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES campaign(id) ON DELETE CASCADE,
    original_url    VARCHAR(2000) NOT NULL,
    short_code      VARCHAR(50) NOT NULL UNIQUE,
    click_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_link_campaign ON tracked_link(campaign_id);
CREATE INDEX idx_link_short ON tracked_link(short_code);

-- Email open events
CREATE TABLE email_open (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_send_id UUID NOT NULL REFERENCES campaign_send(id) ON DELETE CASCADE,
    opened_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    ip_address      INET,
    user_agent      TEXT,
    device_type     VARCHAR(20),
        -- 'desktop', 'mobile', 'tablet', 'unknown'
    email_client    VARCHAR(100),
    is_proxy        BOOLEAN NOT NULL DEFAULT false
        -- Apple Mail Privacy Protection detection
);

CREATE INDEX idx_open_send ON email_open(campaign_send_id);
CREATE INDEX idx_open_time ON email_open(opened_at);

-- Link click events
CREATE TABLE link_click (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tracked_link_id UUID NOT NULL REFERENCES tracked_link(id) ON DELETE CASCADE,
    campaign_send_id UUID NOT NULL REFERENCES campaign_send(id) ON DELETE CASCADE,
    clicked_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    ip_address      INET,
    user_agent      TEXT,
    device_type     VARCHAR(20)
);

CREATE INDEX idx_click_link ON link_click(tracked_link_id);
CREATE INDEX idx_click_send ON link_click(campaign_send_id);
CREATE INDEX idx_click_time ON link_click(clicked_at);

-- Aggregated campaign analytics (materialised for performance)
CREATE TABLE campaign_analytics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES campaign(id) ON DELETE CASCADE UNIQUE,
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
    avg_read_time_seconds NUMERIC(8,2),
    calculated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Accessibility audit results per campaign
CREATE TABLE accessibility_check (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES campaign(id) ON DELETE CASCADE,
    check_type      VARCHAR(50) NOT NULL,
        -- 'alt_text', 'color_contrast', 'heading_structure', 'link_text', 'lang_attr'
    severity        VARCHAR(20) NOT NULL,
        -- 'error', 'warning', 'info'
    message         TEXT NOT NULL,
    element_ref     VARCHAR(255),
    checked_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_access_campaign ON accessibility_check(campaign_id);
```

## AI Features

```sql
-- AI content generation jobs
CREATE TABLE ai_generation_job (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    campaign_id     UUID REFERENCES campaign(id) ON DELETE SET NULL,
    job_type        VARCHAR(50) NOT NULL,
        -- 'draft_newsletter', 'subject_line', 'content_block', 'rewrite'
    input_sources   TEXT[],
        -- e.g. ARRAY['slack:channel-123', 'meeting_notes:456']
    prompt          TEXT,
    generated_content TEXT,
    model_used      VARCHAR(100),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
        -- 'pending', 'running', 'completed', 'failed'
    tokens_used     INTEGER,
    created_by      UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_ai_job_org ON ai_generation_job(organisation_id);
CREATE INDEX idx_ai_job_campaign ON ai_generation_job(campaign_id);

-- Send-time optimisation predictions per subscriber
CREATE TABLE send_time_prediction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscriber_id   UUID NOT NULL REFERENCES subscriber(id) ON DELETE CASCADE,
    best_day_of_week INTEGER,   -- 0=Sunday, 6=Saturday
    best_hour_utc   INTEGER,    -- 0-23
    confidence      NUMERIC(3,2),
    calculated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(subscriber_id)
);

-- AI-powered audience segments
CREATE TABLE smart_segment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    segment_type    VARCHAR(50) NOT NULL,
        -- 'engagement_cluster', 'predicted_churn', 'interest_topic'
    criteria_description TEXT,
    subscriber_count INTEGER NOT NULL DEFAULT 0,
    model_version   VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Junction: subscribers <-> smart segments
CREATE TABLE smart_segment_member (
    smart_segment_id UUID NOT NULL REFERENCES smart_segment(id) ON DELETE CASCADE,
    subscriber_id    UUID NOT NULL REFERENCES subscriber(id) ON DELETE CASCADE,
    score            NUMERIC(5,4),
    assigned_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (smart_segment_id, subscriber_id)
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organisation & Identity | 3 | organisation, app_user, oauth_connection |
| Sending Infrastructure | 2 | sending_domain, delivery_provider |
| Subscribers & Lists | 4 | subscriber, list, subscriber_list, subscriber_preference |
| Templates & Content | 2 | template, media |
| Campaigns & Delivery | 4 | campaign, campaign_content_block, campaign_list, campaign_send |
| Analytics & Engagement | 5 | tracked_link, email_open, link_click, campaign_analytics, accessibility_check |
| AI Features | 4 | ai_generation_job, send_time_prediction, smart_segment, smart_segment_member |
| **Total** | **24** | |

---

## Key Design Decisions

1. **UUID primary keys** — chosen for multi-tenant safety, API-friendliness, and future sharding compatibility. No sequential IDs leak information about record counts.

2. **Separate `campaign_send` per recipient** — enables per-recipient delivery tracking, personalisation variants, and bounce handling. This is the most storage-intensive table but is essential for accurate analytics.

3. **SCIM-aligned subscriber fields** — `employee_id`, `scim_external_id`, `department`, `manager_id` map directly to SCIM 2.0 User and Enterprise User Extension attributes, enabling bidirectional sync with HRIS systems.

4. **Separate `email_open` and `link_click` event tables** — rather than flags on `campaign_send`, these capture multiple events per recipient (re-opens, multiple clicks) with device/client metadata for heatmap generation.

5. **`campaign_analytics` as a materialised aggregate** — pre-computed metrics avoid expensive COUNT/JOIN queries on large event tables. Updated asynchronously after batch event processing.

6. **MJML source stored alongside compiled HTML** — `template.mjml_source` is the source of truth; `compiled_html` is a cache that can be regenerated. This preserves editability while serving fast delivery.

7. **Dynamic lists via `list.dynamic_filter` JSONB** — a controlled use of JSONB for filter criteria in an otherwise normalized model. The application layer evaluates these filters against subscriber attributes.

8. **Consent tracking per preference type** — `subscriber_preference` tracks GDPR lawful basis at the category level (not just a global opt-in), supporting the distinction between mandatory company communications (legitimate interests) and optional newsletters (consent).

9. **Apple Mail Privacy Protection flag** — `email_open.is_proxy` flags proxy-loaded opens, allowing analytics to exclude or discount inflated open rates from Apple MPP.

10. **AI generation jobs as first-class entities** — `ai_generation_job` tracks provenance (input sources, model, tokens) for every AI-generated piece of content, supporting audit trails and cost tracking.
