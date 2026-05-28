# Internal Newsletter Builder — Phased Development Plan

> Project: 165-internal-newsletter-builder · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (Backend) | TypeScript (Node.js) | Email rendering is HTML-centric; MJML, React Email, and GrapesJS are all Node.js/TypeScript-native. TypeScript provides type safety for complex content-block schemas and API contracts. |
| Language (Frontend) | TypeScript (React) | The visual email editor requires a rich interactive UI. React is the ecosystem home for React Email, Easy Email Editor, and GrapesJS integrations. Shared types between frontend and backend. |
| API Framework | Fastify | High-performance Node.js framework with built-in JSON Schema validation, OpenAPI 3.1 generation via `@fastify/swagger`, and first-class TypeScript support. Better suited than Express for structured request validation. |
| Database | PostgreSQL 16 | Required for JSONB columns, GIN indexes, UUID generation, partitioning, and generated columns used in the hybrid relational+JSONB data model (Data Model Suggestion 3). Industry standard for SaaS platforms. |
| ORM / Query Builder | Drizzle ORM | TypeScript-native ORM with strong JSONB support, migration generation, and zero runtime overhead. Works directly with PostgreSQL's native types. |
| Email Rendering | MJML | MIT-licensed responsive email framework; de facto standard for programmatic newsletter template authoring. Handles Outlook VML quirks automatically. Compiles to email-client-compatible HTML. |
| Email Delivery | Amazon SES (primary), SMTP (fallback) | Most cost-effective for high-volume internal newsletter delivery. SMTP fallback supports self-hosted mail servers. Abstracted behind a provider interface. |
| Visual Email Editor | GrapesJS with MJML plugin | Open-source (BSD-3-Clause) drag-and-drop email builder. Embeddable as a React component. MJML plugin generates responsive HTML. More mature than Easy Email Editor. |
| Task Queue | BullMQ (Redis-backed) | Handles async workloads: email sending, AI content generation, HRIS sync, analytics aggregation. BullMQ is the standard Node.js queue with retry, scheduling, and rate limiting. |
| Cache / Queue Backend | Redis 7 | Required by BullMQ for job queues. Also used for session storage, rate limiting, and caching compiled templates. |
| AI / LLM Provider | Anthropic Claude API (via `@anthropic-ai/sdk`) | AI content generation, subject line optimisation, readability scoring, and tone analysis. Prompt caching reduces cost for repeated template patterns. |
| Authentication | Lucia Auth + Oslo | Lightweight auth library for Node.js with session management. Supports password, OAuth 2.0 (RFC 6749), and SAML 2.0 for enterprise SSO. |
| Identity Sync | SCIM 2.0 server (RFC 7643/7644) | Automated employee roster sync from HRIS (Workday, BambooHR, SAP SuccessFactors). Custom SCIM endpoint receives push updates from identity providers. |
| Object Storage | S3-compatible (MinIO for self-hosted) | Media library storage for images, logos, and attachments. S3 API is universally supported. MinIO provides self-hosted S3 compatibility. |
| Frontend Framework | Next.js 15 (App Router) | Server-side rendering for the dashboard, API routes for BFF pattern, built-in image optimisation for media library. Deployed alongside the Fastify API. |
| Testing | Vitest + Playwright | Vitest for unit and integration tests (fast, ESM-native, TypeScript-first). Playwright for E2E browser tests of the visual editor and dashboard. |
| Code Quality | ESLint + Prettier + TypeScript strict | Standard TypeScript linting and formatting. Strict mode catches JSONB access errors at compile time with Zod schemas. |
| Schema Validation | Zod | Runtime validation for API inputs, JSONB field shapes, webhook payloads, and configuration. Generates TypeScript types from schemas. |
| Containerisation | Docker + Docker Compose | Self-hosted deployment target. Multi-stage builds for production images. Compose orchestrates PostgreSQL, Redis, MinIO, and the application. |
| DNS Auth | SPF/DKIM/DMARC verification | Required by Gmail (Feb 2024) and Outlook (May 2025) for bulk senders. Sending domain setup wizard validates DNS records per RFCs 7208, 6376, 7489. |
| Accessibility | WCAG 2.2 checks | Automated accessibility auditing of email content before send. Checks alt text, colour contrast, heading structure per W3C WCAG 2.2. |
| API Documentation | OpenAPI 3.1 (auto-generated) | Generated from Fastify route schemas via `@fastify/swagger`. Enables SDK generation and automation. |
| Package Manager | pnpm | Fast, disk-efficient package manager with workspace support for monorepo structure. |

### Data Model Selection

**Selected: Data Model Suggestion 3 — Hybrid Relational + JSONB**

Rationale: The hybrid model provides the best balance for an MVP-to-production journey. Core entities (subscribers, campaigns, lists) maintain relational integrity with foreign keys, while variable-shape data (HRIS attributes, template block definitions, personalisation rules, engagement profiles, AI metadata) lives in JSONB columns. This reduces the table count to ~11 core tables, supports diverse customer configurations without schema migrations, and avoids the operational complexity of event sourcing. GIN indexes on JSONB columns keep queries performant.

Data Model Suggestion 1 (fully normalized) is too rigid for the variable-shape HRIS and template data. Data Model Suggestion 2 (event-sourced) adds significant complexity that is not justified at MVP stage. The hybrid model can adopt event-sourcing patterns for engagement events later if audit requirements emerge.

### Project Structure

```
internal-newsletter-builder/
├── package.json
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── docker-compose.yml
├── Dockerfile
├── .env.example
├── drizzle.config.ts
├── packages/
│   ├── shared/                     # Shared types, schemas, constants
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── types/
│   │       │   ├── organisation.ts
│   │       │   ├── subscriber.ts
│   │       │   ├── campaign.ts
│   │       │   ├── template.ts
│   │       │   ├── list.ts
│   │       │   ├── analytics.ts
│   │       │   └── ai.ts
│   │       ├── schemas/            # Zod schemas for JSONB validation
│   │       │   ├── subscriber-attributes.ts
│   │       │   ├── template-blocks.ts
│   │       │   ├── campaign-personalisation.ts
│   │       │   ├── engagement-event.ts
│   │       │   └── organisation-settings.ts
│   │       └── constants/
│   │           ├── roles.ts
│   │           ├── statuses.ts
│   │           └── event-types.ts
│   ├── api/                        # Fastify API server
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── server.ts
│   │       ├── config.ts
│   │       ├── db/
│   │       │   ├── schema.ts       # Drizzle schema definitions
│   │       │   ├── connection.ts
│   │       │   └── migrations/
│   │       ├── routes/
│   │       │   ├── auth.ts
│   │       │   ├── organisations.ts
│   │       │   ├── subscribers.ts
│   │       │   ├── lists.ts
│   │       │   ├── templates.ts
│   │       │   ├── campaigns.ts
│   │       │   ├── analytics.ts
│   │       │   ├── media.ts
│   │       │   ├── sending-domains.ts
│   │       │   ├── ai.ts
│   │       │   └── scim.ts
│   │       ├── services/
│   │       │   ├── email-renderer.ts
│   │       │   ├── email-sender.ts
│   │       │   ├── delivery-provider.ts
│   │       │   ├── subscriber-sync.ts
│   │       │   ├── analytics-aggregator.ts
│   │       │   ├── accessibility-checker.ts
│   │       │   ├── ai-content-generator.ts
│   │       │   ├── send-time-optimiser.ts
│   │       │   └── link-tracker.ts
│   │       ├── workers/
│   │       │   ├── campaign-sender.ts
│   │       │   ├── analytics-worker.ts
│   │       │   ├── scim-sync-worker.ts
│   │       │   └── ai-generation-worker.ts
│   │       ├── middleware/
│   │       │   ├── auth.ts
│   │       │   ├── org-context.ts
│   │       │   └── rate-limit.ts
│   │       └── plugins/
│   │           ├── swagger.ts
│   │           └── cors.ts
│   └── web/                        # Next.js frontend
│       ├── package.json
│       ├── tsconfig.json
│       ├── next.config.ts
│       └── src/
│           ├── app/
│           │   ├── layout.tsx
│           │   ├── page.tsx
│           │   ├── (auth)/
│           │   │   ├── login/
│           │   │   └── register/
│           │   └── (dashboard)/
│           │       ├── layout.tsx
│           │       ├── campaigns/
│           │       ├── templates/
│           │       ├── subscribers/
│           │       ├── lists/
│           │       ├── analytics/
│           │       ├── media/
│           │       └── settings/
│           ├── components/
│           │   ├── editor/          # GrapesJS email editor wrapper
│           │   ├── analytics/       # Charts, heatmaps
│           │   ├── subscribers/     # Import, list management
│           │   ├── campaigns/       # Campaign builder, scheduler
│           │   └── ui/              # Shared UI components
│           ├── lib/
│           │   ├── api-client.ts
│           │   └── hooks/
│           └── styles/
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/
│       ├── templates/
│       ├── subscribers/
│       └── campaigns/
└── scripts/
    ├── seed.ts
    ├── migrate.ts
    └── verify-dns.ts
```

---

## Phase 1: Foundation & Project Scaffolding

### Purpose
Establish the monorepo structure, database connection, core data model, and development tooling. After this phase, the project builds, lints, and connects to PostgreSQL with the schema applied. No user-facing features yet, but the foundation supports all subsequent phases.

### Tasks

#### 1.1 — Monorepo Setup and Tooling

**What**: Initialise the pnpm workspace monorepo with shared, api, and web packages, and configure TypeScript, ESLint, Prettier, and Vitest.

**Design**:

Root `pnpm-workspace.yaml`:
```yaml
packages:
  - "packages/*"
```

Root `tsconfig.base.json`:
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "dist",
    "baseUrl": ".",
    "paths": {
      "@newsletter/shared/*": ["packages/shared/src/*"],
      "@newsletter/api/*": ["packages/api/src/*"]
    }
  }
}
```

`.env.example`:
```bash
# Database
DATABASE_URL=postgresql://newsletter:newsletter@localhost:5432/newsletter
# Redis
REDIS_URL=redis://localhost:6379
# Object Storage
S3_ENDPOINT=http://localhost:9000
S3_BUCKET=newsletter-media
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
# Auth
SESSION_SECRET=change-me-in-production
# AI
ANTHROPIC_API_KEY=sk-ant-...
# Email
DEFAULT_FROM_EMAIL=newsletter@example.com
DEFAULT_FROM_NAME=Company Newsletter
```

**Testing**:
- `Unit: pnpm install completes without errors`
- `Unit: pnpm build compiles all three packages without TypeScript errors`
- `Unit: pnpm lint passes with zero warnings on initial codebase`
- `Unit: pnpm test runs Vitest and reports zero tests (baseline)`

---

#### 1.2 — Database Schema and Migrations

**What**: Define the Drizzle ORM schema matching Data Model Suggestion 3 (Hybrid Relational + JSONB) and generate the initial migration.

**Design**:

Drizzle schema in `packages/api/src/db/schema.ts`:

```typescript
import { pgTable, uuid, varchar, text, boolean, integer, bigint, timestamp, jsonb, char, uniqueIndex, index, primaryKey } from "drizzle-orm/pg-core";
import { sql } from "drizzle-orm";

// --- Organisation & Users ---

export const organisation = pgTable("organisation", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: varchar("name", { length: 255 }).notNull(),
  slug: varchar("slug", { length: 100 }).notNull().unique(),
  plan: varchar("plan", { length: 50 }).notNull().default("free"),
  settings: jsonb("settings").notNull().default({}),
  integrations: jsonb("integrations").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export const appUser = pgTable("app_user", {
  id: uuid("id").primaryKey().defaultRandom(),
  organisationId: uuid("organisation_id").notNull().references(() => organisation.id, { onDelete: "cascade" }),
  email: varchar("email", { length: 320 }).notNull(),
  displayName: varchar("display_name", { length: 255 }),
  passwordHash: varchar("password_hash", { length: 255 }),
  role: varchar("role", { length: 50 }).notNull().default("editor"),
  isActive: boolean("is_active").notNull().default(true),
  preferences: jsonb("preferences").notNull().default({}),
  lastLoginAt: timestamp("last_login_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex("uq_app_user_org_email").on(table.organisationId, table.email),
  index("idx_app_user_org").on(table.organisationId),
]);

export const sendingDomain = pgTable("sending_domain", {
  id: uuid("id").primaryKey().defaultRandom(),
  organisationId: uuid("organisation_id").notNull().references(() => organisation.id, { onDelete: "cascade" }),
  domain: varchar("domain", { length: 255 }).notNull(),
  dnsConfig: jsonb("dns_config").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex("uq_sending_domain_org_domain").on(table.organisationId, table.domain),
]);

// --- Subscribers & Lists ---

export const subscriber = pgTable("subscriber", {
  id: uuid("id").primaryKey().defaultRandom(),
  organisationId: uuid("organisation_id").notNull().references(() => organisation.id, { onDelete: "cascade" }),
  email: varchar("email", { length: 320 }).notNull(),
  firstName: varchar("first_name", { length: 100 }),
  lastName: varchar("last_name", { length: 100 }),
  department: varchar("department", { length: 255 }),
  jobTitle: varchar("job_title", { length: 255 }),
  countryCode: char("country_code", { length: 2 }),
  timezone: varchar("timezone", { length: 50 }),
  employeeId: varchar("employee_id", { length: 100 }),
  status: varchar("status", { length: 20 }).notNull().default("active"),
  source: varchar("source", { length: 50 }).notNull().default("manual"),
  customAttributes: jsonb("custom_attributes").notNull().default({}),
  preferences: jsonb("preferences").notNull().default({}),
  engagementProfile: jsonb("engagement_profile").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex("uq_subscriber_org_email").on(table.organisationId, table.email),
  index("idx_subscriber_org").on(table.organisationId),
  index("idx_subscriber_status").on(table.organisationId, table.status),
  index("idx_subscriber_dept").on(table.organisationId, table.department),
]);

export const list = pgTable("list", {
  id: uuid("id").primaryKey().defaultRandom(),
  organisationId: uuid("organisation_id").notNull().references(() => organisation.id, { onDelete: "cascade" }),
  name: varchar("name", { length: 255 }).notNull(),
  description: text("description"),
  type: varchar("type", { length: 20 }).notNull().default("manual"),
  filterRules: jsonb("filter_rules"),
  subscriberCount: integer("subscriber_count").notNull().default(0),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_list_org").on(table.organisationId),
]);

export const subscriberList = pgTable("subscriber_list", {
  subscriberId: uuid("subscriber_id").notNull().references(() => subscriber.id, { onDelete: "cascade" }),
  listId: uuid("list_id").notNull().references(() => list.id, { onDelete: "cascade" }),
  addedAt: timestamp("added_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  primaryKey({ columns: [table.subscriberId, table.listId] }),
  index("idx_sub_list_list").on(table.listId),
]);

// --- Templates & Media ---

export const template = pgTable("template", {
  id: uuid("id").primaryKey().defaultRandom(),
  organisationId: uuid("organisation_id").notNull().references(() => organisation.id, { onDelete: "cascade" }),
  name: varchar("name", { length: 255 }).notNull(),
  category: varchar("category", { length: 100 }),
  version: integer("version").notNull().default(1),
  blocks: jsonb("blocks").notNull().default([]),
  settings: jsonb("settings").notNull().default({}),
  compiledHtml: text("compiled_html"),
  thumbnailUrl: varchar("thumbnail_url", { length: 500 }),
  isSystem: boolean("is_system").notNull().default(false),
  createdBy: uuid("created_by").references(() => appUser.id),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_template_org").on(table.organisationId),
]);

export const media = pgTable("media", {
  id: uuid("id").primaryKey().defaultRandom(),
  organisationId: uuid("organisation_id").notNull().references(() => organisation.id, { onDelete: "cascade" }),
  filename: varchar("filename", { length: 255 }).notNull(),
  contentType: varchar("content_type", { length: 100 }).notNull(),
  fileSize: bigint("file_size", { mode: "number" }).notNull(),
  storagePath: varchar("storage_path", { length: 500 }).notNull(),
  altText: varchar("alt_text", { length: 500 }),
  dimensions: jsonb("dimensions"),
  uploadedBy: uuid("uploaded_by").references(() => appUser.id),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_media_org").on(table.organisationId),
]);

// --- Campaigns ---

export const campaign = pgTable("campaign", {
  id: uuid("id").primaryKey().defaultRandom(),
  organisationId: uuid("organisation_id").notNull().references(() => organisation.id, { onDelete: "cascade" }),
  name: varchar("name", { length: 255 }).notNull(),
  subject: varchar("subject", { length: 500 }).notNull(),
  previewText: varchar("preview_text", { length: 500 }),
  fromName: varchar("from_name", { length: 255 }),
  fromEmail: varchar("from_email", { length: 320 }),
  replyTo: varchar("reply_to", { length: 320 }),
  templateId: uuid("template_id").references(() => template.id),
  status: varchar("status", { length: 20 }).notNull().default("draft"),
  contentBlocks: jsonb("content_blocks").notNull().default([]),
  personalisation: jsonb("personalisation").notNull().default({}),
  targetLists: uuid("target_lists").array().notNull().default(sql`'{}'`),
  aiMetadata: jsonb("ai_metadata"),
  accessibilityReport: jsonb("accessibility_report"),
  scheduledAt: timestamp("scheduled_at", { withTimezone: true }),
  sentAt: timestamp("sent_at", { withTimezone: true }),
  completedAt: timestamp("completed_at", { withTimezone: true }),
  createdBy: uuid("created_by").references(() => appUser.id),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_campaign_org").on(table.organisationId),
  index("idx_campaign_status").on(table.organisationId, table.status),
]);

// --- Analytics ---

export const engagementEvent = pgTable("engagement_event", {
  id: uuid("id").primaryKey().defaultRandom(),
  organisationId: uuid("organisation_id").notNull(),
  campaignId: uuid("campaign_id").notNull().references(() => campaign.id, { onDelete: "cascade" }),
  subscriberId: uuid("subscriber_id").notNull().references(() => subscriber.id, { onDelete: "cascade" }),
  eventType: varchar("event_type", { length: 30 }).notNull(),
  occurredAt: timestamp("occurred_at", { withTimezone: true }).notNull().defaultNow(),
  eventData: jsonb("event_data").notNull().default({}),
}, (table) => [
  index("idx_engage_campaign").on(table.campaignId, table.eventType),
  index("idx_engage_subscriber").on(table.subscriberId, table.occurredAt),
  index("idx_engage_time").on(table.occurredAt),
]);

export const campaignAnalytics = pgTable("campaign_analytics", {
  campaignId: uuid("campaign_id").primaryKey().references(() => campaign.id, { onDelete: "cascade" }),
  totalRecipients: integer("total_recipients").notNull().default(0),
  totalSent: integer("total_sent").notNull().default(0),
  totalDelivered: integer("total_delivered").notNull().default(0),
  totalBounced: integer("total_bounced").notNull().default(0),
  uniqueOpens: integer("unique_opens").notNull().default(0),
  totalOpens: integer("total_opens").notNull().default(0),
  uniqueClicks: integer("unique_clicks").notNull().default(0),
  totalClicks: integer("total_clicks").notNull().default(0),
  openRate: varchar("open_rate", { length: 10 }),
  clickRate: varchar("click_rate", { length: 10 }),
  bounceRate: varchar("bounce_rate", { length: 10 }),
  breakdown: jsonb("breakdown").notNull().default({}),
  calculatedAt: timestamp("calculated_at", { withTimezone: true }).notNull().defaultNow(),
});
```

Database connection in `packages/api/src/db/connection.ts`:
```typescript
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import * as schema from "./schema";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
export const db = drizzle(pool, { schema });
```

**Testing**:
- `Unit: Drizzle schema compiles without TypeScript errors`
- `Integration: drizzle-kit generate produces a valid SQL migration file`
- `Integration: drizzle-kit migrate applies the migration to a fresh PostgreSQL database`
- `Integration: all tables exist with correct columns after migration (query information_schema.columns)`
- `Unit: JSONB default values are correct (empty object or empty array per column)`

---

#### 1.3 — Docker Compose Development Environment

**What**: Create a Docker Compose file that runs PostgreSQL, Redis, and MinIO for local development, plus a Dockerfile for the application.

**Design**:

`docker-compose.yml`:
```yaml
version: "3.8"
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: newsletter
      POSTGRES_PASSWORD: newsletter
      POSTGRES_DB: newsletter
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U newsletter"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - miniodata:/data

volumes:
  pgdata:
  miniodata:
```

Multi-stage `Dockerfile`:
```dockerfile
FROM node:22-alpine AS base
RUN corepack enable && corepack prepare pnpm@latest --activate

FROM base AS deps
WORKDIR /app
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY packages/shared/package.json packages/shared/
COPY packages/api/package.json packages/api/
COPY packages/web/package.json packages/web/
RUN pnpm install --frozen-lockfile

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --from=deps /app/packages/shared/node_modules ./packages/shared/node_modules
COPY --from=deps /app/packages/api/node_modules ./packages/api/node_modules
COPY --from=deps /app/packages/web/node_modules ./packages/web/node_modules
COPY . .
RUN pnpm build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/packages/api/dist ./packages/api/dist
COPY --from=builder /app/packages/shared/dist ./packages/shared/dist
COPY --from=builder /app/packages/web/.next ./packages/web/.next
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3000 3001
CMD ["node", "packages/api/dist/server.js"]
```

**Testing**:
- `Integration: docker compose up -d starts all three services without errors`
- `Integration: PostgreSQL accepts connections on port 5432 with the configured credentials`
- `Integration: Redis responds to PING on port 6379`
- `Integration: MinIO console is accessible on port 9001`
- `Integration: docker compose down removes all containers cleanly`

---

#### 1.4 — Fastify API Server Bootstrap

**What**: Create the Fastify server with health check, CORS, Swagger/OpenAPI plugin, and request validation infrastructure.

**Design**:

`packages/api/src/config.ts`:
```typescript
import { z } from "zod";

const ConfigSchema = z.object({
  PORT: z.coerce.number().default(3001),
  HOST: z.string().default("0.0.0.0"),
  DATABASE_URL: z.string(),
  REDIS_URL: z.string().default("redis://localhost:6379"),
  SESSION_SECRET: z.string().min(32),
  S3_ENDPOINT: z.string().default("http://localhost:9000"),
  S3_BUCKET: z.string().default("newsletter-media"),
  S3_ACCESS_KEY: z.string().default("minioadmin"),
  S3_SECRET_KEY: z.string().default("minioadmin"),
  ANTHROPIC_API_KEY: z.string().optional(),
  DEFAULT_FROM_EMAIL: z.string().email().default("newsletter@example.com"),
  DEFAULT_FROM_NAME: z.string().default("Company Newsletter"),
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
});

export type Config = z.infer<typeof ConfigSchema>;
export const config = ConfigSchema.parse(process.env);
```

`packages/api/src/server.ts`:
```typescript
import Fastify from "fastify";
import cors from "@fastify/cors";
import swagger from "@fastify/swagger";
import swaggerUi from "@fastify/swagger-ui";
import { config } from "./config";

export async function buildApp() {
  const app = Fastify({ logger: true });

  await app.register(cors, { origin: true, credentials: true });

  await app.register(swagger, {
    openapi: {
      info: {
        title: "Internal Newsletter Builder API",
        version: "1.0.0",
        description: "API for managing internal newsletters, subscribers, and analytics",
      },
      servers: [{ url: `http://localhost:${config.PORT}` }],
    },
  });

  await app.register(swaggerUi, { routePrefix: "/docs" });

  app.get("/health", async () => ({
    status: "ok",
    timestamp: new Date().toISOString(),
    version: "1.0.0",
  }));

  return app;
}

async function start() {
  const app = await buildApp();
  await app.listen({ port: config.PORT, host: config.HOST });
}

start();
```

**Testing**:
- `Unit: buildApp() returns a Fastify instance without throwing`
- `Integration: GET /health returns 200 with status "ok" and ISO timestamp`
- `Integration: GET /docs returns the Swagger UI HTML page`
- `Integration: GET /docs/json returns a valid OpenAPI 3.1 document`
- `Unit: ConfigSchema.parse with missing DATABASE_URL throws ZodError`
- `Unit: ConfigSchema.parse with valid env returns typed config object`

---

## Phase 2: Authentication & Multi-Tenancy

### Purpose
Implement user authentication (password-based with session management), organisation creation, role-based access control, and the multi-tenant context middleware. After this phase, users can register, log in, create organisations, and all subsequent API routes are scoped to the authenticated user's organisation.

### Tasks

#### 2.1 — User Registration and Login

**What**: Implement password-based registration and login with Lucia session management.

**Design**:

Session management uses Lucia Auth with PostgreSQL adapter:

```typescript
// packages/api/src/services/auth.ts
import { Lucia } from "lucia";
import { DrizzlePostgreSQLAdapter } from "@lucia-auth/adapter-drizzle";
import { db } from "../db/connection";
import { appUser } from "../db/schema";

// Session table added to schema
export const session = pgTable("user_session", {
  id: varchar("id", { length: 255 }).primaryKey(),
  userId: uuid("user_id").notNull().references(() => appUser.id, { onDelete: "cascade" }),
  expiresAt: timestamp("expires_at", { withTimezone: true }).notNull(),
});

const adapter = new DrizzlePostgreSQLAdapter(db, session, appUser);

export const lucia = new Lucia(adapter, {
  sessionCookie: { attributes: { secure: process.env.NODE_ENV === "production" } },
  getUserAttributes: (attributes) => ({
    email: attributes.email,
    displayName: attributes.displayName,
    role: attributes.role,
    organisationId: attributes.organisationId,
  }),
});
```

API routes:

```typescript
// POST /api/auth/register
interface RegisterRequest {
  email: string;         // max 320 chars, valid email
  password: string;      // min 8 chars
  displayName: string;   // max 255 chars
  organisationName: string; // max 255 chars
}
interface RegisterResponse {
  user: { id: string; email: string; displayName: string; role: string };
  organisation: { id: string; name: string; slug: string };
}

// POST /api/auth/login
interface LoginRequest {
  email: string;
  password: string;
}
interface LoginResponse {
  user: { id: string; email: string; displayName: string; role: string; organisationId: string };
}

// POST /api/auth/logout
// Requires authenticated session
// Returns 204 No Content

// GET /api/auth/me
// Returns current user profile or 401
```

Password hashing uses oslo/password with Argon2id:
```typescript
import { Argon2id } from "oslo/password";
const hasher = new Argon2id();
// Register: hash = await hasher.hash(password)
// Login: valid = await hasher.verify(storedHash, password)
```

**Testing**:
- `Unit: password hashing produces a different hash each time (salt verification)`
- `Unit: password verification succeeds with correct password`
- `Unit: password verification fails with incorrect password`
- `Integration: POST /api/auth/register with valid data creates user and organisation, returns 201`
- `Integration: POST /api/auth/register with duplicate email returns 409 Conflict`
- `Integration: POST /api/auth/register with password < 8 chars returns 400`
- `Integration: POST /api/auth/login with valid credentials returns 200 with session cookie`
- `Integration: POST /api/auth/login with wrong password returns 401`
- `Integration: GET /api/auth/me with valid session returns user profile`
- `Integration: GET /api/auth/me without session returns 401`
- `Integration: POST /api/auth/logout invalidates session cookie`

---

#### 2.2 — Organisation Management and Multi-Tenant Middleware

**What**: Implement the organisation context middleware that scopes all data access to the authenticated user's organisation, plus organisation settings CRUD.

**Design**:

```typescript
// packages/api/src/middleware/org-context.ts
import { FastifyRequest, FastifyReply } from "fastify";

declare module "fastify" {
  interface FastifyRequest {
    organisationId: string;
    userId: string;
    userRole: "owner" | "admin" | "editor" | "viewer";
  }
}

export async function orgContextMiddleware(
  request: FastifyRequest,
  reply: FastifyReply
): Promise<void> {
  const session = await lucia.validateSession(request.cookies.session_id);
  if (!session || !session.user) {
    return reply.status(401).send({ error: "Unauthorized" });
  }
  request.organisationId = session.user.organisationId;
  request.userId = session.user.id;
  request.userRole = session.user.role;
}

// Role guard factory
export function requireRole(...roles: Array<"owner" | "admin" | "editor" | "viewer">) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    if (!roles.includes(request.userRole)) {
      return reply.status(403).send({ error: "Forbidden" });
    }
  };
}
```

Organisation settings API:

```typescript
// GET /api/organisation
// Returns current organisation details and settings

// PATCH /api/organisation
// Requires role: owner | admin
interface UpdateOrganisationRequest {
  name?: string;
  settings?: {
    defaultFromName?: string;
    defaultFromEmail?: string;
    branding?: { primaryColor?: string; logoUrl?: string };
    aiEnabled?: boolean;
    defaultTimezone?: string;
  };
}
```

Organisation settings Zod schema:
```typescript
// packages/shared/src/schemas/organisation-settings.ts
import { z } from "zod";

export const OrganisationSettingsSchema = z.object({
  defaultFromName: z.string().max(255).optional(),
  defaultFromEmail: z.string().email().optional(),
  branding: z.object({
    primaryColor: z.string().regex(/^#[0-9a-fA-F]{6}$/).optional(),
    logoUrl: z.string().url().optional(),
  }).optional(),
  aiEnabled: z.boolean().optional(),
  defaultTimezone: z.string().optional(),
  deliveryProvider: z.enum(["ses", "sendgrid", "smtp"]).optional(),
}).strict();
```

**Testing**:
- `Unit: orgContextMiddleware with valid session sets organisationId on request`
- `Unit: orgContextMiddleware without session returns 401`
- `Unit: requireRole("admin") with editor role returns 403`
- `Unit: requireRole("admin", "editor") with editor role passes`
- `Integration: GET /api/organisation returns org details for authenticated user`
- `Integration: PATCH /api/organisation as owner updates settings`
- `Integration: PATCH /api/organisation as viewer returns 403`
- `Integration: all data queries filter by organisationId (verify with two orgs in test DB)`

---

#### 2.3 — Invite Users to Organisation

**What**: Allow organisation owners and admins to invite users by email with a specific role.

**Design**:

```typescript
// POST /api/organisation/invites
// Requires role: owner | admin
interface InviteRequest {
  email: string;
  role: "admin" | "editor" | "viewer";
}
interface InviteResponse {
  id: string;
  email: string;
  role: string;
  invitedBy: string;
  expiresAt: string;  // 7 days from creation
}

// GET /api/organisation/invites
// List pending invitations

// DELETE /api/organisation/invites/:id
// Cancel a pending invitation

// POST /api/auth/accept-invite
// Accept invitation with invite token
interface AcceptInviteRequest {
  token: string;
  password: string;
  displayName: string;
}
```

Invite token is a 64-character random hex string stored with Argon2id hash. Raw token sent via email; only hash persists in the database.

**Testing**:
- `Unit: invite token generation produces 64-character hex strings`
- `Integration: POST /api/organisation/invites creates invitation and returns invite details`
- `Integration: POST /api/organisation/invites as viewer returns 403`
- `Integration: POST /api/organisation/invites with duplicate email returns 409`
- `Integration: POST /api/auth/accept-invite with valid token creates user in organisation`
- `Integration: POST /api/auth/accept-invite with expired token returns 410 Gone`
- `Integration: POST /api/auth/accept-invite with invalid token returns 404`
- `Integration: DELETE /api/organisation/invites/:id cancels pending invite`

---

## Phase 3: Subscriber Management & HRIS Integration

### Purpose
Implement the subscriber lifecycle: manual creation, CSV import, SCIM 2.0 sync from HRIS systems, mailing list management (manual and dynamic), and email preference/consent tracking for GDPR compliance (Articles 6, 21). After this phase, organisations can populate their employee directory and build audience segments.

### Tasks

#### 3.1 — Subscriber CRUD and CSV Import

**What**: Build subscriber management API with create, read, update, list (with filtering), and bulk CSV import.

**Design**:

```typescript
// packages/shared/src/types/subscriber.ts
export interface Subscriber {
  id: string;
  organisationId: string;
  email: string;
  firstName: string | null;
  lastName: string | null;
  department: string | null;
  jobTitle: string | null;
  countryCode: string | null;   // ISO 3166-1 alpha-2
  timezone: string | null;       // IANA timezone
  employeeId: string | null;
  status: "active" | "unsubscribed" | "bounced" | "complained";
  source: "manual" | "scim" | "csv_import" | "api";
  customAttributes: Record<string, unknown>;
  preferences: SubscriberPreferences;
  engagementProfile: EngagementProfile;
  createdAt: string;
  updatedAt: string;
}

export interface SubscriberPreferences {
  categories?: Record<string, {
    optedIn: boolean;
    basis?: "legitimate_interests" | "consent";
    consentedAt?: string;
    withdrawnAt?: string;
  }>;
  frequency?: "daily" | "weekly" | "monthly";
  format?: "html" | "plaintext";
}

export interface EngagementProfile {
  engagementScore?: number;      // 0.0 - 1.0
  segment?: string;
  avgOpenRate?: number;
  avgClickRate?: number;
  bestSendHourUtc?: number;      // 0-23
  bestSendDay?: number;          // 0=Sunday, 6=Saturday
  preferredDevice?: string;
  preferredClient?: string;
  lastOpenedAt?: string;
  topicsOfInterest?: string[];
  churnRisk?: number;            // 0.0 - 1.0
}
```

API endpoints:

```typescript
// POST /api/subscribers
interface CreateSubscriberRequest {
  email: string;
  firstName?: string;
  lastName?: string;
  department?: string;
  jobTitle?: string;
  countryCode?: string;
  timezone?: string;
  employeeId?: string;
  customAttributes?: Record<string, unknown>;
}

// GET /api/subscribers?page=1&limit=50&status=active&department=Engineering&search=jane
interface SubscriberListResponse {
  data: Subscriber[];
  pagination: { page: number; limit: number; total: number; totalPages: number };
}

// GET /api/subscribers/:id
// PATCH /api/subscribers/:id
// DELETE /api/subscribers/:id (soft-delete: sets status to "unsubscribed")

// POST /api/subscribers/import
// Multipart form upload: CSV file
// CSV columns: email (required), first_name, last_name, department, job_title, country_code, timezone, employee_id
// Returns: { imported: number; skipped: number; errors: Array<{ row: number; error: string }> }
```

CSV import uses a streaming parser to handle large files without loading the full file into memory:
```typescript
import { parse } from "csv-parse";
// Stream CSV rows, validate each with Zod, upsert into database in batches of 100
```

**Testing**:
- `Unit: CreateSubscriberSchema validates email format and country_code length`
- `Unit: CreateSubscriberSchema rejects invalid country code "USA" (must be 2 chars)`
- `Integration: POST /api/subscribers creates subscriber with generated UUID`
- `Integration: POST /api/subscribers with duplicate email in same org returns 409`
- `Integration: GET /api/subscribers returns paginated list scoped to organisation`
- `Integration: GET /api/subscribers?department=Engineering filters correctly`
- `Integration: GET /api/subscribers?search=jane matches firstName, lastName, and email`
- `Integration: PATCH /api/subscribers/:id updates only provided fields`
- `Integration: DELETE /api/subscribers/:id sets status to "unsubscribed"`
- `Integration: POST /api/subscribers/import with valid CSV creates subscribers`
- `Integration: POST /api/subscribers/import with invalid email in row 3 returns error for row 3 and imports the rest`
- `Fixture-based: import test CSV file with 100 rows including duplicates and invalid data`

---

#### 3.2 — Mailing List Management (Manual and Dynamic)

**What**: Implement mailing lists with manual membership and dynamic filter-based membership evaluation.

**Design**:

```typescript
// packages/shared/src/schemas/list-filter.ts
import { z } from "zod";

const FilterConditionSchema = z.object({
  field: z.string(),       // "department", "country_code", "custom_attributes.division", "engagement_profile.engagement_score"
  op: z.enum(["eq", "ne", "in", "not_in", "gt", "gte", "lt", "lte", "contains", "exists"]),
  value: z.union([z.string(), z.number(), z.boolean()]).optional(),
  values: z.array(z.union([z.string(), z.number()])).optional(),
});

export const ListFilterSchema = z.object({
  operator: z.enum(["AND", "OR"]).default("AND"),
  conditions: z.array(FilterConditionSchema).min(1),
});

export type ListFilter = z.infer<typeof ListFilterSchema>;
```

Dynamic list evaluation:
```typescript
// packages/api/src/services/list-evaluator.ts
export async function evaluateDynamicList(
  organisationId: string,
  filterRules: ListFilter
): Promise<string[]> {
  // Build a Drizzle query from the filter conditions
  // Standard columns (department, country_code, status) use typed column filters
  // custom_attributes.* and engagement_profile.* use JSONB containment operators
  // Returns array of subscriber IDs matching all conditions
}
```

API endpoints:
```typescript
// POST /api/lists
interface CreateListRequest {
  name: string;
  description?: string;
  type: "manual" | "dynamic";
  filterRules?: ListFilter;   // Required when type is "dynamic"
}

// GET /api/lists
// GET /api/lists/:id
// PATCH /api/lists/:id
// DELETE /api/lists/:id

// POST /api/lists/:id/subscribers     (manual lists only)
interface AddSubscribersRequest {
  subscriberIds: string[];
}

// DELETE /api/lists/:id/subscribers/:subscriberId  (manual lists only)

// GET /api/lists/:id/subscribers?page=1&limit=50
// For manual lists: returns members
// For dynamic lists: evaluates filter and returns matching subscribers

// POST /api/lists/:id/preview
// Dry-run a dynamic list filter, returns count and sample subscribers
```

**Testing**:
- `Unit: ListFilterSchema validates AND/OR operator`
- `Unit: ListFilterSchema rejects empty conditions array`
- `Unit: evaluateDynamicList with department="Engineering" returns only engineering subscribers`
- `Unit: evaluateDynamicList with nested custom_attributes filter uses JSONB containment`
- `Integration: POST /api/lists creates manual list`
- `Integration: POST /api/lists with type="dynamic" stores filterRules`
- `Integration: POST /api/lists/:id/subscribers adds members to manual list`
- `Integration: POST /api/lists/:id/subscribers on a dynamic list returns 400`
- `Integration: GET /api/lists/:id/subscribers on dynamic list evaluates filter in real-time`
- `Integration: POST /api/lists/:id/preview returns count without persisting`
- `Integration: subscriber count is updated when members are added/removed`

---

#### 3.3 — SCIM 2.0 Endpoint for HRIS Sync

**What**: Implement a SCIM 2.0 server endpoint (RFC 7643/7644) that receives push updates from identity providers (Azure AD, Okta, Workday) to automatically sync the subscriber list.

**Design**:

SCIM 2.0 endpoints:
```typescript
// All SCIM endpoints are under /scim/v2/ and authenticated via Bearer token

// GET /scim/v2/Users
// SCIM list/filter endpoint
// Supports: ?filter=userName eq "jane@company.com"
// Returns: SCIM ListResponse with Resources[]

// GET /scim/v2/Users/:id
// Returns single SCIM User resource

// POST /scim/v2/Users
// Create subscriber from SCIM User payload
interface ScimUserResource {
  schemas: ["urn:ietf:params:scim:schemas:core:2.0:User"];
  userName: string;           // maps to subscriber.email
  name: {
    givenName: string;        // maps to subscriber.firstName
    familyName: string;       // maps to subscriber.lastName
  };
  emails: Array<{ value: string; primary: boolean }>;
  active: boolean;            // maps to subscriber.status
  title?: string;             // maps to subscriber.jobTitle
  "urn:ietf:params:scim:schemas:extension:enterprise:2.0:User"?: {
    department?: string;      // maps to subscriber.department
    employeeNumber?: string;  // maps to subscriber.employeeId
    manager?: { value: string };
  };
}

// PUT /scim/v2/Users/:id
// Full replace of subscriber attributes from SCIM

// PATCH /scim/v2/Users/:id
// Partial update per SCIM Patch Operations (RFC 7644 Section 3.5.2)

// DELETE /scim/v2/Users/:id
// Deactivate subscriber (set status = "unsubscribed")
```

SCIM-to-subscriber mapping function:
```typescript
function scimUserToSubscriber(scimUser: ScimUserResource): Partial<Subscriber> {
  const enterprise = scimUser["urn:ietf:params:scim:schemas:extension:enterprise:2.0:User"];
  return {
    email: scimUser.userName,
    firstName: scimUser.name?.givenName,
    lastName: scimUser.name?.familyName,
    jobTitle: scimUser.title,
    department: enterprise?.department,
    employeeId: enterprise?.employeeNumber,
    status: scimUser.active ? "active" : "unsubscribed",
    source: "scim",
    customAttributes: { scimExternalId: scimUser.id },
  };
}
```

SCIM Bearer token authentication: a per-organisation token stored in `organisation.integrations.scim.bearerToken` (Argon2id hash).

**Testing**:
- `Unit: scimUserToSubscriber maps all standard SCIM attributes correctly`
- `Unit: scimUserToSubscriber maps enterprise extension attributes (department, employeeNumber)`
- `Unit: scimUserToSubscriber sets source to "scim"`
- `Integration: POST /scim/v2/Users creates subscriber with source="scim"`
- `Integration: POST /scim/v2/Users with invalid Bearer token returns 401`
- `Integration: PUT /scim/v2/Users/:id updates subscriber attributes`
- `Integration: PATCH /scim/v2/Users/:id with "op":"replace" updates specific fields`
- `Integration: DELETE /scim/v2/Users/:id sets subscriber status to "unsubscribed"`
- `Integration: GET /scim/v2/Users?filter=userName eq "jane@company.com" returns matching user`
- `Integration: GET /scim/v2/Users returns SCIM ListResponse with totalResults`
- `Fixture-based: SCIM Azure AD test payload creates subscriber with correct attribute mapping`
- `Fixture-based: SCIM Okta test payload creates subscriber with correct attribute mapping`

---

#### 3.4 — Email Preferences and GDPR Consent Tracking

**What**: Implement subscriber preference management with GDPR-compliant consent recording (Article 6 lawful basis, Article 21 right to object).

**Design**:

```typescript
// GET /api/subscribers/:id/preferences
// Returns the subscriber's preferences JSONB

// PATCH /api/subscribers/:id/preferences
interface UpdatePreferencesRequest {
  categories?: Record<string, {
    optedIn: boolean;
    basis?: "legitimate_interests" | "consent";
  }>;
  frequency?: "daily" | "weekly" | "monthly";
  format?: "html" | "plaintext";
}

// POST /api/unsubscribe
// One-click unsubscribe per RFC 8058
// Accepts: List-Unsubscribe-Post header format
interface UnsubscribeRequest {
  token: string;           // JWT containing subscriberId and campaignId
  category?: string;       // Optional: unsubscribe from specific category only
}
```

Consent tracking is embedded in the subscriber.preferences JSONB. Every change records the timestamp:
```typescript
// When opting in with consent basis:
preferences.categories["team_news"] = {
  optedIn: true,
  basis: "consent",
  consentedAt: "2026-05-25T10:00:00Z"
};

// When withdrawing consent:
preferences.categories["team_news"] = {
  optedIn: false,
  basis: "consent",
  consentedAt: "2026-01-15T10:00:00Z",
  withdrawnAt: "2026-05-25T10:00:00Z"
};
```

One-click unsubscribe token is a JWT:
```typescript
import { createSigner, createVerifier } from "fast-jwt";
// Payload: { sub: subscriberId, cid: campaignId, cat?: "team_news", exp: 30 days }
```

Every email includes `List-Unsubscribe` and `List-Unsubscribe-Post` headers per RFC 8058:
```
List-Unsubscribe: <https://newsletter.example.com/api/unsubscribe?token=xxx>
List-Unsubscribe-Post: List-Unsubscribe=One-Click
```

**Testing**:
- `Unit: UnsubscribeTokenSchema validates JWT structure`
- `Unit: consent recording stores timestamp and lawful basis`
- `Unit: consent withdrawal preserves original consentedAt and adds withdrawnAt`
- `Integration: PATCH /api/subscribers/:id/preferences updates category opt-in`
- `Integration: POST /api/unsubscribe with valid token sets optedIn=false for category`
- `Integration: POST /api/unsubscribe with expired token returns 410`
- `Integration: POST /api/unsubscribe without category unsubscribes from all`
- `Integration: List-Unsubscribe header is present in sent emails (verified in Phase 5)`
- `Integration: subscriber with withdrawn consent is excluded from campaign sends`

---

## Phase 4: Template System & Email Rendering

### Purpose
Build the MJML-based email template system with a content-block architecture, the visual drag-and-drop editor (GrapesJS), personalisation token rendering, and WCAG 2.2 accessibility checking. After this phase, users can create, edit, and preview email templates with dynamic content blocks.

### Tasks

#### 4.1 — Template CRUD and Block Schema

**What**: Implement the template management API with JSONB-based content blocks and MJML compilation.

**Design**:

```typescript
// packages/shared/src/schemas/template-blocks.ts
import { z } from "zod";

export const TemplateBlockSchema = z.object({
  id: z.string(),
  type: z.enum(["header", "text", "image", "button", "divider", "social", "video", "spacer", "columns"]),
  mjml: z.string(),
  personalisationTokens: z.array(z.string()).default([]),
  aiEditable: z.boolean().default(false),
  wcagRequiresAlt: z.boolean().default(false),
  tracked: z.boolean().default(false),
});

export const TemplateSettingsSchema = z.object({
  fontFamily: z.string().default("Arial, sans-serif"),
  backgroundColor: z.string().default("#ffffff"),
  textColor: z.string().default("#333333"),
  linkColor: z.string().default("#1a73e8"),
  maxWidth: z.string().default("600px"),
});

export type TemplateBlock = z.infer<typeof TemplateBlockSchema>;
export type TemplateSettings = z.infer<typeof TemplateSettingsSchema>;
```

MJML compilation service:
```typescript
// packages/api/src/services/email-renderer.ts
import mjml2html from "mjml";

export interface RenderResult {
  html: string;
  errors: Array<{ line: number; message: string; tagName: string }>;
}

export function compileTemplate(
  blocks: TemplateBlock[],
  settings: TemplateSettings
): RenderResult {
  const mjmlSource = wrapBlocks(blocks, settings);
  const result = mjml2html(mjmlSource, { validationLevel: "strict" });
  return { html: result.html, errors: result.errors };
}

function wrapBlocks(blocks: TemplateBlock[], settings: TemplateSettings): string {
  const blocksMjml = blocks.map((b) => b.mjml).join("\n");
  return `
    <mjml>
      <mj-head>
        <mj-attributes>
          <mj-all font-family="${settings.fontFamily}" color="${settings.textColor}" />
          <mj-text font-size="16px" line-height="1.5" />
        </mj-attributes>
      </mj-head>
      <mj-body background-color="${settings.backgroundColor}" width="${settings.maxWidth}">
        ${blocksMjml}
      </mj-body>
    </mjml>
  `;
}
```

API endpoints:
```typescript
// POST /api/templates
interface CreateTemplateRequest {
  name: string;
  category?: string;
  blocks: TemplateBlock[];
  settings?: TemplateSettings;
}

// GET /api/templates?category=company_update
// GET /api/templates/:id
// PATCH /api/templates/:id
// DELETE /api/templates/:id

// POST /api/templates/:id/preview
// Compiles MJML blocks and returns rendered HTML
interface PreviewResponse {
  html: string;
  errors: Array<{ line: number; message: string }>;
}

// POST /api/templates/:id/duplicate
// Creates a copy of the template with "(copy)" appended to the name
```

**Testing**:
- `Unit: TemplateBlockSchema validates all block types`
- `Unit: TemplateBlockSchema rejects unknown block type "carousel"`
- `Unit: compileTemplate with a header and text block returns valid HTML`
- `Unit: compileTemplate with invalid MJML returns errors array`
- `Unit: wrapBlocks applies settings (font, colors, width) to the MJML wrapper`
- `Integration: POST /api/templates creates template with blocks stored as JSONB`
- `Integration: GET /api/templates/:id returns template with blocks array`
- `Integration: POST /api/templates/:id/preview returns compiled HTML`
- `Integration: PATCH /api/templates/:id updates blocks without changing id`
- `Integration: POST /api/templates/:id/duplicate creates copy with incremented version`
- `Fixture-based: compile "company-update" template fixture with 5 blocks into valid HTML`

---

#### 4.2 — Visual Email Editor (GrapesJS Integration)

**What**: Integrate GrapesJS with MJML plugin as a React component for drag-and-drop email editing in the frontend.

**Design**:

```typescript
// packages/web/src/components/editor/EmailEditor.tsx
import { useCallback, useEffect, useRef } from "react";
import grapesjs, { Editor } from "grapesjs";
import grapesjsMjml from "grapesjs-mjml";
import type { TemplateBlock, TemplateSettings } from "@newsletter/shared/types/template";

interface EmailEditorProps {
  blocks: TemplateBlock[];
  settings: TemplateSettings;
  onSave: (blocks: TemplateBlock[], mjml: string) => void;
  onChange?: (isDirty: boolean) => void;
}

export function EmailEditor({ blocks, settings, onSave, onChange }: EmailEditorProps) {
  const editorRef = useRef<Editor | null>(null);
  const containerRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!containerRef.current) return;

    const editor = grapesjs.init({
      container: containerRef.current,
      plugins: [grapesjsMjml],
      pluginsOpts: { [grapesjsMjml]: {} },
      storageManager: false,
      // Custom block categories for newsletter components
    });

    // Load initial MJML content from blocks
    const mjml = blocksToMjml(blocks, settings);
    editor.setComponents(mjml);

    editorRef.current = editor;

    return () => { editor.destroy(); };
  }, []);

  const handleSave = useCallback(() => {
    if (!editorRef.current) return;
    const mjml = editorRef.current.getHtml(); // MJML output
    const parsedBlocks = mjmlToBlocks(mjml);
    onSave(parsedBlocks, mjml);
  }, [onSave]);

  return (
    <div>
      <div ref={containerRef} style={{ height: "80vh" }} />
      <button onClick={handleSave}>Save Template</button>
    </div>
  );
}
```

The editor provides custom MJML blocks for common newsletter patterns:
- Company header with logo
- Text section with personalisation tokens
- Image with alt text prompt (WCAG 2.2)
- Call-to-action button with link tracking
- Social media links
- Divider/spacer

**Testing**:
- `E2E (Playwright): editor loads and renders GrapesJS canvas`
- `E2E (Playwright): drag text block onto canvas creates a new section`
- `E2E (Playwright): edit text content and save produces updated MJML`
- `E2E (Playwright): add image block prompts for alt text`
- `E2E (Playwright): save template triggers onSave with correct block structure`
- `Unit: blocksToMjml converts block array to valid MJML string`
- `Unit: mjmlToBlocks parses MJML string back into block array`

---

#### 4.3 — Personalisation Token Engine

**What**: Implement merge-tag rendering that replaces personalisation tokens (e.g., `{{first_name}}`, `{{department}}`) with subscriber-specific values before sending.

**Design**:

```typescript
// packages/api/src/services/personalisation-engine.ts
import Handlebars from "handlebars";

export interface PersonalisationContext {
  subscriber: {
    email: string;
    firstName: string | null;
    lastName: string | null;
    department: string | null;
    jobTitle: string | null;
    countryCode: string | null;
    customAttributes: Record<string, unknown>;
  };
  organisation: {
    name: string;
    branding: { primaryColor: string; logoUrl: string };
  };
  campaign: {
    name: string;
    unsubscribeUrl: string;
    webViewUrl: string;
  };
}

export function renderPersonalised(
  html: string,
  context: PersonalisationContext,
  variants?: CampaignPersonalisation
): string {
  // 1. Apply variant overrides based on subscriber attributes
  let content = html;
  if (variants?.variants) {
    for (const variant of variants.variants) {
      if (matchesConditions(context.subscriber, variant.conditions)) {
        content = applyBlockOverrides(content, variant.blockOverrides);
        break; // First matching variant wins
      }
    }
  }
  // 2. Compile and render Handlebars template
  const template = Handlebars.compile(content);
  return template(context);
}

function matchesConditions(
  subscriber: PersonalisationContext["subscriber"],
  conditions: Record<string, string>
): boolean {
  return Object.entries(conditions).every(([key, value]) => {
    if (key.startsWith("customAttributes.")) {
      const attr = key.replace("customAttributes.", "");
      return subscriber.customAttributes[attr] === value;
    }
    return (subscriber as any)[key] === value;
  });
}
```

Supported tokens:
- `{{subscriber.firstName}}` — subscriber first name
- `{{subscriber.lastName}}` — subscriber last name
- `{{subscriber.department}}` — subscriber department
- `{{subscriber.customAttributes.division}}` — custom HRIS attribute
- `{{organisation.name}}` — organisation name
- `{{campaign.unsubscribeUrl}}` — one-click unsubscribe link (RFC 8058)
- `{{campaign.webViewUrl}}` — "view in browser" link

**Testing**:
- `Unit: renderPersonalised replaces {{subscriber.firstName}} with "Jane"`
- `Unit: renderPersonalised with null firstName renders empty string`
- `Unit: renderPersonalised replaces nested {{subscriber.customAttributes.division}}`
- `Unit: renderPersonalised applies variant overrides when department matches`
- `Unit: renderPersonalised uses default content when no variant matches`
- `Unit: renderPersonalised with unsubscribeUrl produces valid URL`
- `Unit: undefined token renders empty string (no raw {{}} in output)`
- `Fixture-based: render "company-update" template with 5 different subscriber contexts`

---

#### 4.4 — WCAG 2.2 Accessibility Checker

**What**: Automated accessibility auditing of email HTML before send, checking alt text, colour contrast, heading structure, and link text per WCAG 2.2.

**Design**:

```typescript
// packages/api/src/services/accessibility-checker.ts
import { JSDOM } from "jsdom";

export interface AccessibilityIssue {
  type: "alt_text" | "color_contrast" | "heading_structure" | "link_text" | "lang_attr" | "table_structure";
  severity: "error" | "warning" | "info";
  message: string;
  element?: string;   // CSS selector or tag reference
}

export interface AccessibilityReport {
  passed: boolean;
  issues: AccessibilityIssue[];
  checkedAt: string;
}

export function checkAccessibility(html: string): AccessibilityReport {
  const dom = new JSDOM(html);
  const doc = dom.window.document;
  const issues: AccessibilityIssue[] = [];

  // 1. Alt text: all <img> must have non-empty alt attribute
  doc.querySelectorAll("img").forEach((img, i) => {
    if (!img.getAttribute("alt")) {
      issues.push({
        type: "alt_text",
        severity: "error",
        message: `Image ${i + 1} is missing alt text`,
        element: `img:nth-of-type(${i + 1})`,
      });
    }
  });

  // 2. Link text: no <a> elements with empty text content or "click here"
  doc.querySelectorAll("a").forEach((a, i) => {
    const text = a.textContent?.trim();
    if (!text) {
      issues.push({
        type: "link_text",
        severity: "error",
        message: `Link ${i + 1} has no text content`,
        element: `a:nth-of-type(${i + 1})`,
      });
    } else if (["click here", "read more", "here"].includes(text.toLowerCase())) {
      issues.push({
        type: "link_text",
        severity: "warning",
        message: `Link ${i + 1} uses non-descriptive text "${text}"`,
        element: `a:nth-of-type(${i + 1})`,
      });
    }
  });

  // 3. Heading structure: headings should not skip levels
  // 4. Lang attribute: <html> should have lang attribute
  // 5. Table structure: data tables need headers

  return {
    passed: issues.filter((i) => i.severity === "error").length === 0,
    issues,
    checkedAt: new Date().toISOString(),
  };
}
```

API endpoint:
```typescript
// POST /api/campaigns/:id/accessibility-check
// Runs the accessibility checker on the campaign's compiled HTML
// Stores result in campaign.accessibility_report JSONB
// Returns AccessibilityReport
```

**Testing**:
- `Unit: checkAccessibility with img missing alt text reports error`
- `Unit: checkAccessibility with all imgs having alt text reports no alt_text errors`
- `Unit: checkAccessibility with "click here" link reports warning`
- `Unit: checkAccessibility with descriptive link text passes`
- `Unit: checkAccessibility with skipped heading level (h1 then h3) reports warning`
- `Unit: checkAccessibility with missing lang attribute reports warning`
- `Integration: POST /api/campaigns/:id/accessibility-check stores report in campaign row`
- `Fixture-based: check accessibility of "company-update" template HTML fixture`

---

#### 4.5 — Media Library

**What**: Upload, store, and serve images for use in email templates via S3-compatible object storage.

**Design**:

```typescript
// POST /api/media
// Multipart upload with file and alt_text
// Validates: file type (image/png, image/jpeg, image/gif, image/webp, image/svg+xml)
// Validates: file size (max 5MB)
// Stores in S3 bucket under: org-{organisationId}/{uuid}.{ext}
// Returns: { id, filename, contentType, fileSize, storagePath, altText, dimensions, url }

// GET /api/media?page=1&limit=50
// Returns paginated list of media for the organisation

// GET /api/media/:id
// Returns media metadata and a signed URL for the file

// DELETE /api/media/:id
// Deletes from S3 and database

// GET /api/media/:id/file
// Redirects to signed S3 URL (or proxies the file for privacy)
```

```typescript
// packages/api/src/services/media-storage.ts
import { S3Client, PutObjectCommand, DeleteObjectCommand, GetObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import sharp from "sharp";

export class MediaStorage {
  private s3: S3Client;
  private bucket: string;

  constructor(config: { endpoint: string; accessKey: string; secretKey: string; bucket: string }) {
    this.s3 = new S3Client({
      endpoint: config.endpoint,
      credentials: { accessKeyId: config.accessKey, secretAccessKey: config.secretKey },
      forcePathStyle: true, // Required for MinIO
    });
    this.bucket = config.bucket;
  }

  async upload(file: Buffer, key: string, contentType: string): Promise<void> {
    await this.s3.send(new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      Body: file,
      ContentType: contentType,
    }));
  }

  async getSignedUrl(key: string, expiresIn: number = 3600): Promise<string> {
    return getSignedUrl(this.s3, new GetObjectCommand({
      Bucket: this.bucket,
      Key: key,
    }), { expiresIn });
  }

  async delete(key: string): Promise<void> {
    await this.s3.send(new DeleteObjectCommand({
      Bucket: this.bucket,
      Key: key,
    }));
  }

  async getDimensions(file: Buffer): Promise<{ width: number; height: number }> {
    const metadata = await sharp(file).metadata();
    return { width: metadata.width ?? 0, height: metadata.height ?? 0 };
  }
}
```

**Testing**:
- `Unit: MediaStorage.upload stores file at correct S3 key`
- `Unit: MediaStorage.getSignedUrl returns a URL with expiry`
- `Unit: MediaStorage.getDimensions returns width and height for a PNG`
- `Integration: POST /api/media uploads image and creates media record`
- `Integration: POST /api/media with file > 5MB returns 413`
- `Integration: POST /api/media with non-image content type returns 400`
- `Integration: GET /api/media returns paginated list for organisation`
- `Integration: DELETE /api/media/:id removes file from S3 and database`
- `Integration (mocked S3): upload failure returns 500 with error message`

---

## Phase 5: Campaign Builder & Email Delivery

### Purpose
Build the campaign creation workflow: compose a newsletter from a template, target audience lists, schedule or send immediately, and deliver via SMTP/SES with link tracking, bounce handling, and RFC 8058 one-click unsubscribe headers. After this phase, the core product loop is complete: create template, build campaign, send to subscribers, and track delivery.

### Tasks

#### 5.1 — Campaign CRUD and Content Assembly

**What**: Implement campaign creation, content block customisation from templates, target list selection, and scheduling.

**Design**:

```typescript
// POST /api/campaigns
interface CreateCampaignRequest {
  name: string;
  subject: string;
  previewText?: string;
  templateId: string;        // Copies blocks from template
  targetLists: string[];     // List IDs
  fromName?: string;         // Overrides org default
  fromEmail?: string;        // Overrides org default
  replyTo?: string;
}
// On creation: copy template.blocks into campaign.content_blocks (snapshot)

// GET /api/campaigns?status=draft&page=1&limit=20
// GET /api/campaigns/:id
// PATCH /api/campaigns/:id
// DELETE /api/campaigns/:id (only if status is "draft")

// PATCH /api/campaigns/:id/content
interface UpdateContentRequest {
  contentBlocks: TemplateBlock[];     // Updated block array
  personalisation?: CampaignPersonalisation;
}

// POST /api/campaigns/:id/schedule
interface ScheduleRequest {
  scheduledAt: string;    // ISO 8601 datetime, must be in the future
}

// POST /api/campaigns/:id/send-now
// Transitions campaign to "sending" and enqueues the send job

// POST /api/campaigns/:id/preview
// Returns rendered HTML for a specific subscriber (or with sample data)
interface PreviewRequest {
  subscriberId?: string;   // Render for this subscriber
}

// POST /api/campaigns/:id/cancel
// Only if status is "scheduled" — transitions to "cancelled"
```

Campaign state machine:
```
draft → scheduled → sending → sent
draft → sending → sent
scheduled → cancelled
draft → cancelled
```

**Testing**:
- `Unit: campaign creation copies template blocks into content_blocks`
- `Unit: campaign creation validates targetLists are valid list IDs`
- `Integration: POST /api/campaigns creates campaign in "draft" status`
- `Integration: PATCH /api/campaigns/:id/content updates content blocks`
- `Integration: POST /api/campaigns/:id/schedule sets scheduledAt and status to "scheduled"`
- `Integration: POST /api/campaigns/:id/schedule with past datetime returns 400`
- `Integration: POST /api/campaigns/:id/send-now transitions to "sending"`
- `Integration: POST /api/campaigns/:id/send-now on non-draft campaign returns 409`
- `Integration: POST /api/campaigns/:id/cancel transitions scheduled campaign to "cancelled"`
- `Integration: POST /api/campaigns/:id/preview renders HTML with subscriber data`
- `Integration: DELETE /api/campaigns/:id on sent campaign returns 409`

---

#### 5.2 — Email Delivery Provider Abstraction

**What**: Build a delivery provider interface with implementations for Amazon SES and raw SMTP, supporting SPF/DKIM/DMARC verified sending domains (RFCs 7208, 6376, 7489).

**Design**:

```typescript
// packages/api/src/services/delivery-provider.ts
export interface EmailMessage {
  to: string;
  from: { name: string; email: string };
  replyTo?: string;
  subject: string;
  html: string;
  text: string;           // Plain text alternative (RFC 2045 MIME multipart)
  headers: Record<string, string>;  // List-Unsubscribe, List-Unsubscribe-Post
}

export interface DeliveryResult {
  messageId: string;       // Provider's message ID
  status: "sent" | "failed";
  error?: string;
}

export interface DeliveryProvider {
  send(message: EmailMessage): Promise<DeliveryResult>;
  sendBatch(messages: EmailMessage[]): Promise<DeliveryResult[]>;
  verifyDomain(domain: string): Promise<{ spf: boolean; dkim: boolean; dmarc: boolean }>;
}

// packages/api/src/services/ses-provider.ts
import { SESv2Client, SendEmailCommand } from "@aws-sdk/client-sesv2";

export class SesProvider implements DeliveryProvider {
  private client: SESv2Client;

  constructor(config: { region: string; accessKeyId: string; secretAccessKey: string }) {
    this.client = new SESv2Client(config);
  }

  async send(message: EmailMessage): Promise<DeliveryResult> {
    const command = new SendEmailCommand({
      FromEmailAddress: `${message.from.name} <${message.from.email}>`,
      Destination: { ToAddresses: [message.to] },
      Content: {
        Simple: {
          Subject: { Data: message.subject },
          Body: {
            Html: { Data: message.html },
            Text: { Data: message.text },
          },
          Headers: Object.entries(message.headers).map(([name, value]) => ({
            Name: name,
            Value: value,
          })),
        },
      },
    });
    const response = await this.client.send(command);
    return { messageId: response.MessageId!, status: "sent" };
  }

  async sendBatch(messages: EmailMessage[]): Promise<DeliveryResult[]> {
    // Send in parallel with concurrency limit of 10
    // Retry failed sends up to 3 times with exponential backoff
  }
}

// packages/api/src/services/smtp-provider.ts
import nodemailer from "nodemailer";

export class SmtpProvider implements DeliveryProvider {
  private transporter: nodemailer.Transporter;

  constructor(config: { host: string; port: number; secure: boolean; auth: { user: string; pass: string } }) {
    this.transporter = nodemailer.createTransport(config);
  }

  async send(message: EmailMessage): Promise<DeliveryResult> {
    const info = await this.transporter.sendMail({
      from: `"${message.from.name}" <${message.from.email}>`,
      to: message.to,
      replyTo: message.replyTo,
      subject: message.subject,
      html: message.html,
      text: message.text,
      headers: message.headers,
    });
    return { messageId: info.messageId, status: "sent" };
  }
}
```

**Testing**:
- `Unit: SesProvider.send constructs correct SendEmailCommand`
- `Unit: SmtpProvider.send constructs correct nodemailer message`
- `Integration (mocked SES): send returns messageId and status "sent"`
- `Integration (mocked SES): send with invalid from address returns status "failed"`
- `Integration (mocked SMTP): send delivers message via SMTP transport`
- `Unit: sendBatch respects concurrency limit of 10`
- `Unit: sendBatch retries failed sends up to 3 times`
- `Unit: email includes List-Unsubscribe and List-Unsubscribe-Post headers`
- `Unit: email includes both HTML and plain text body (MIME multipart)`

---

#### 5.3 — Link Tracking

**What**: Replace links in email HTML with tracking URLs, record clicks, and redirect to the original URL.

**Design**:

```typescript
// packages/api/src/services/link-tracker.ts
export interface TrackedLink {
  id: string;
  campaignId: string;
  originalUrl: string;
  trackingCode: string;   // 8-char base62
  position: number;       // Order of link in email (for heatmap positioning)
}

export function rewriteLinks(
  html: string,
  campaignId: string,
  baseUrl: string
): { html: string; links: TrackedLink[] } {
  const links: TrackedLink[] = [];
  let position = 0;

  // Parse HTML, find all <a href="..."> tags
  // Skip: mailto:, tel:, #anchor links
  // Replace each href with: ${baseUrl}/t/${trackingCode}
  // Store mapping: trackingCode -> originalUrl

  return { html: rewrittenHtml, links };
}

// GET /t/:trackingCode
// 1. Record click event in engagement_event table
// 2. 302 redirect to original URL
```

The tracking endpoint is separate from the API server for performance and to avoid authentication middleware:
```typescript
// Tracking endpoint (lightweight, no auth)
app.get("/t/:code", async (request, reply) => {
  const { code } = request.params;
  const link = await findTrackedLink(code);
  if (!link) return reply.status(404).send();

  // Record click event asynchronously (don't block redirect)
  recordClickEvent(link, request).catch(console.error);

  return reply.redirect(302, link.originalUrl);
});
```

**Testing**:
- `Unit: rewriteLinks replaces http/https links with tracking URLs`
- `Unit: rewriteLinks skips mailto: links`
- `Unit: rewriteLinks skips tel: links`
- `Unit: rewriteLinks skips anchor (#) links`
- `Unit: rewriteLinks assigns sequential positions to tracked links`
- `Unit: tracking code is 8 characters of base62`
- `Integration: GET /t/:code with valid code redirects to original URL`
- `Integration: GET /t/:code records engagement_event with event_type="clicked"`
- `Integration: GET /t/:code with unknown code returns 404`
- `Integration: click event includes device_type and ip from request headers`

---

#### 5.4 — Campaign Send Worker (BullMQ)

**What**: Build the background worker that processes campaign sends: resolves audience lists, renders personalised emails, sends via the delivery provider, and records delivery events.

**Design**:

```typescript
// packages/api/src/workers/campaign-sender.ts
import { Worker, Queue, Job } from "bullmq";

export const campaignQueue = new Queue("campaign-send", { connection: redisConnection });

interface CampaignSendJob {
  campaignId: string;
  organisationId: string;
}

export const campaignWorker = new Worker("campaign-send", async (job: Job<CampaignSendJob>) => {
  const { campaignId, organisationId } = job.data;

  // 1. Load campaign with content blocks and personalisation rules
  const campaign = await loadCampaign(campaignId);

  // 2. Resolve target lists to subscriber list
  //    - Manual lists: fetch subscriber_list memberships
  //    - Dynamic lists: evaluate filter_rules against subscriber table
  //    - Deduplicate across lists
  //    - Exclude subscribers with status != "active"
  //    - Exclude subscribers who have opted out of the campaign's category
  const subscribers = await resolveAudience(campaign.targetLists, organisationId);

  // 3. Compile MJML content blocks to HTML
  const baseHtml = compileTemplate(campaign.contentBlocks, templateSettings);

  // 4. Rewrite links for tracking
  const { html: trackedHtml, links } = rewriteLinks(baseHtml, campaignId, trackingBaseUrl);
  await saveTrackedLinks(links);

  // 5. For each subscriber (batched, concurrent):
  //    a. Render personalised HTML (merge tags + variant overrides)
  //    b. Generate plain text version (html-to-text)
  //    c. Add unsubscribe headers (RFC 8058)
  //    d. Send via delivery provider
  //    e. Record engagement_event with event_type="sent"
  //    f. Update job progress

  // 6. Update campaign status to "sent" with completedAt timestamp
  // 7. Trigger analytics aggregation job

}, {
  concurrency: 5,           // Process 5 campaigns concurrently
  limiter: { max: 50, duration: 1000 },  // Rate limit: 50 emails/sec
});
```

Open tracking pixel:
```typescript
// Inject a 1x1 tracking pixel at the end of each email:
// <img src="${trackingBaseUrl}/o/${sendId}" width="1" height="1" alt="" style="display:block" />

// GET /o/:sendId
// Records engagement_event with event_type="opened"
// Returns: 1x1 transparent GIF
```

**Testing**:
- `Unit: resolveAudience deduplicates subscribers across multiple lists`
- `Unit: resolveAudience excludes unsubscribed subscribers`
- `Unit: resolveAudience excludes subscribers opted out of campaign category`
- `Unit: resolveAudience evaluates dynamic list filters correctly`
- `Integration (mocked provider): campaign send worker sends emails to all resolved subscribers`
- `Integration (mocked provider): each sent email has personalised content`
- `Integration (mocked provider): each email includes List-Unsubscribe header`
- `Integration (mocked provider): engagement_event records created for each send`
- `Integration (mocked provider): campaign status updated to "sent" after completion`
- `Integration: GET /o/:sendId returns 1x1 GIF and records open event`
- `Integration: GET /o/:sendId with Apple Mail Privacy Protection detected sets is_proxy flag`
- `Integration: scheduled campaign triggers send job at the scheduled time`
- `Unit: rate limiter caps sending at 50 emails per second`

---

#### 5.5 — Sending Domain Setup & DNS Verification

**What**: Build the sending domain wizard that guides users through SPF, DKIM, and DMARC DNS record configuration and verifies compliance per RFCs 7208, 6376, 7489.

**Design**:

```typescript
// POST /api/sending-domains
interface CreateSendingDomainRequest {
  domain: string;   // e.g., "company.com"
}
// Returns: required DNS records for SPF, DKIM, DMARC

// GET /api/sending-domains
// GET /api/sending-domains/:id

// POST /api/sending-domains/:id/verify
// Performs DNS lookup and validates SPF, DKIM, DMARC records
// Updates dns_config JSONB with verification results
interface VerifyResponse {
  spf: { verified: boolean; record: string | null; error?: string };
  dkim: { verified: boolean; selector: string; error?: string };
  dmarc: { verified: boolean; policy: string | null; error?: string };
  allVerified: boolean;
}
```

DNS verification service:
```typescript
// packages/api/src/services/dns-verifier.ts
import { resolveTxt } from "dns/promises";

export async function verifySPF(domain: string): Promise<{ verified: boolean; record: string | null }> {
  const records = await resolveTxt(domain);
  const spfRecord = records.flat().find((r) => r.startsWith("v=spf1"));
  return { verified: !!spfRecord, record: spfRecord ?? null };
}

export async function verifyDKIM(domain: string, selector: string): Promise<{ verified: boolean }> {
  const dkimDomain = `${selector}._domainkey.${domain}`;
  try {
    const records = await resolveTxt(dkimDomain);
    return { verified: records.flat().some((r) => r.includes("v=DKIM1")) };
  } catch {
    return { verified: false };
  }
}

export async function verifyDMARC(domain: string): Promise<{ verified: boolean; policy: string | null }> {
  try {
    const records = await resolveTxt(`_dmarc.${domain}`);
    const dmarcRecord = records.flat().find((r) => r.startsWith("v=DMARC1"));
    const policy = dmarcRecord?.match(/p=(\w+)/)?.[1] ?? null;
    return { verified: !!dmarcRecord, policy };
  } catch {
    return { verified: false, policy: null };
  }
}
```

**Testing**:
- `Unit (mocked DNS): verifySPF with valid SPF record returns verified=true`
- `Unit (mocked DNS): verifySPF with no SPF record returns verified=false`
- `Unit (mocked DNS): verifyDKIM with valid DKIM record returns verified=true`
- `Unit (mocked DNS): verifyDMARC with p=reject returns policy="reject"`
- `Integration: POST /api/sending-domains creates domain with initial dns_config`
- `Integration: POST /api/sending-domains/:id/verify updates dns_config with results`
- `Integration: campaign send with unverified domain returns 400 (blocked)`

---

## Phase 6: Analytics & Engagement Tracking

### Purpose
Build the analytics pipeline: process delivery webhook events (bounces, complaints), aggregate per-campaign metrics, compute engagement heatmaps and department breakdowns, and surface analytics via dashboard API endpoints. After this phase, users can see detailed engagement data for every campaign they send.

### Tasks

#### 6.1 — Webhook Event Processing (Bounces, Complaints)

**What**: Implement webhook endpoints for SES and SMTP delivery notifications (bounces, complaints, deliveries) and record them as engagement events.

**Design**:

```typescript
// POST /api/webhooks/ses
// Amazon SES SNS notification handler
// Handles: Bounce, Complaint, Delivery notification types
interface SesNotification {
  Type: "Notification";
  Message: string;  // JSON string containing SES event
}

interface SesBounceEvent {
  notificationType: "Bounce";
  bounce: {
    bounceType: "Permanent" | "Transient";
    bouncedRecipients: Array<{ emailAddress: string }>;
  };
  mail: { messageId: string };
}

interface SesComplaintEvent {
  notificationType: "Complaint";
  complaint: {
    complainedRecipients: Array<{ emailAddress: string }>;
    complaintFeedbackType: string;
  };
  mail: { messageId: string };
}

// POST /api/webhooks/smtp
// Generic SMTP bounce handler (for self-hosted SMTP setups)
```

Event recording:
```typescript
// For each webhook event:
// 1. Verify webhook signature (SES SNS signature verification)
// 2. Map provider message ID to campaign_send
// 3. Record engagement_event:
//    - bounce: event_type="bounced", event_data={bounce_type, reason}
//    - complaint: event_type="complained", event_data={feedback_type}
//    - delivery: event_type="delivered"
// 4. For hard bounces: update subscriber.status = "bounced"
// 5. For complaints: update subscriber.status = "complained"
```

**Testing**:
- `Unit: SES bounce notification maps to engagement_event with correct type`
- `Unit: SES complaint notification updates subscriber status to "complained"`
- `Unit: hard bounce updates subscriber status to "bounced"`
- `Unit: soft bounce does NOT update subscriber status`
- `Integration (mocked SNS): POST /api/webhooks/ses with valid signature processes event`
- `Integration (mocked SNS): POST /api/webhooks/ses with invalid signature returns 403`
- `Fixture-based: process SES bounce notification fixture`
- `Fixture-based: process SES complaint notification fixture`

---

#### 6.2 — Analytics Aggregation Worker

**What**: Build the background worker that periodically aggregates engagement events into the campaign_analytics table, including department/device/client breakdowns stored in the breakdown JSONB.

**Design**:

```typescript
// packages/api/src/workers/analytics-worker.ts
import { Worker, Queue } from "bullmq";

export const analyticsQueue = new Queue("analytics-aggregation", { connection: redisConnection });

interface AggregationJob {
  campaignId: string;
}

export const analyticsWorker = new Worker("analytics-aggregation", async (job: Job<AggregationJob>) => {
  const { campaignId } = job.data;

  // 1. Count events by type
  const counts = await db.execute(sql`
    SELECT event_type, COUNT(*) as count,
           COUNT(DISTINCT subscriber_id) as unique_count
    FROM engagement_event
    WHERE campaign_id = ${campaignId}
    GROUP BY event_type
  `);

  // 2. Build breakdown by department
  const deptBreakdown = await db.execute(sql`
    SELECT s.department,
           COUNT(*) FILTER (WHERE ee.event_type = 'sent') as sent,
           COUNT(DISTINCT ee.subscriber_id) FILTER (WHERE ee.event_type = 'opened') as opens,
           COUNT(DISTINCT ee.subscriber_id) FILTER (WHERE ee.event_type = 'clicked') as clicks
    FROM engagement_event ee
    JOIN subscriber s ON ee.subscriber_id = s.id
    WHERE ee.campaign_id = ${campaignId}
    GROUP BY s.department
  `);

  // 3. Build breakdown by device type
  const deviceBreakdown = await db.execute(sql`
    SELECT event_data->>'device_type' as device_type, COUNT(*) as count
    FROM engagement_event
    WHERE campaign_id = ${campaignId}
      AND event_type = 'opened'
    GROUP BY event_data->>'device_type'
  `);

  // 4. Build breakdown by email client
  // 5. Build breakdown by hour of day
  // 6. Build link click ranking

  // 7. Upsert into campaign_analytics
  await db.insert(campaignAnalytics).values({
    campaignId,
    totalRecipients: totalSent,
    totalSent,
    totalDelivered,
    totalBounced,
    uniqueOpens,
    totalOpens,
    uniqueClicks,
    totalClicks,
    openRate: (uniqueOpens / totalDelivered).toFixed(4),
    clickRate: (uniqueClicks / totalDelivered).toFixed(4),
    bounceRate: (totalBounced / totalSent).toFixed(4),
    breakdown: { byDepartment: deptBreakdown, byDevice: deviceBreakdown, byClient: clientBreakdown, byHour: hourBreakdown, linkClicks: linkRanking },
    calculatedAt: new Date(),
  }).onConflictDoUpdate({ target: campaignAnalytics.campaignId, set: { /* all fields */ } });
});
```

**Testing**:
- `Unit: aggregation correctly counts unique opens (deduplicates by subscriber)`
- `Unit: aggregation excludes proxy opens from unique_opens when is_proxy is true`
- `Unit: open rate calculation handles zero delivered (returns 0, not NaN)`
- `Integration: analytics worker produces correct totals for fixture campaign with 100 events`
- `Integration: department breakdown groups events by subscriber.department`
- `Integration: device breakdown extracts device_type from event_data JSONB`
- `Integration: subsequent aggregation runs update existing campaign_analytics row`

---

#### 6.3 — Analytics API Endpoints

**What**: Expose analytics data via REST endpoints for the dashboard: campaign overview, engagement heatmaps, department comparison, and time-series data.

**Design**:

```typescript
// GET /api/campaigns/:id/analytics
// Returns campaign_analytics row with breakdown data
interface CampaignAnalyticsResponse {
  totalRecipients: number;
  totalSent: number;
  totalDelivered: number;
  totalBounced: number;
  uniqueOpens: number;
  totalOpens: number;
  uniqueClicks: number;
  totalClicks: number;
  openRate: number;
  clickRate: number;
  bounceRate: number;
  breakdown: {
    byDepartment: Record<string, { sent: number; opens: number; clicks: number }>;
    byDevice: Record<string, number>;
    byClient: Record<string, number>;
    byHour: Record<string, number>;
    linkClicks: Array<{ url: string; position: number; clicks: number }>;
  };
}

// GET /api/campaigns/:id/analytics/subscribers?page=1&limit=50
// Per-subscriber engagement data for a campaign
interface SubscriberEngagementResponse {
  data: Array<{
    subscriberId: string;
    email: string;
    firstName: string;
    department: string;
    sentAt: string | null;
    deliveredAt: string | null;
    firstOpenedAt: string | null;
    openCount: number;
    clickCount: number;
    deviceType: string | null;
  }>;
  pagination: { page: number; limit: number; total: number };
}

// GET /api/analytics/overview?period=30d
// Organisation-level analytics summary across all campaigns
interface OverviewResponse {
  totalCampaigns: number;
  totalEmailsSent: number;
  avgOpenRate: number;
  avgClickRate: number;
  avgBounceRate: number;
  topCampaigns: Array<{ id: string; name: string; openRate: number }>;
  trendData: Array<{ date: string; sent: number; opens: number; clicks: number }>;
}
```

**Testing**:
- `Integration: GET /api/campaigns/:id/analytics returns aggregated metrics`
- `Integration: GET /api/campaigns/:id/analytics/subscribers returns per-subscriber data`
- `Integration: GET /api/analytics/overview returns organisation-level summary`
- `Integration: overview trendData covers the requested period (30 days)`
- `Integration: analytics endpoints are scoped to the authenticated user's organisation`
- `Unit: analytics response correctly calculates rates from raw counts`

---

## Phase 7: AI Content Generation

### Purpose
Integrate Claude AI for newsletter content drafting, subject line optimisation, readability scoring, and tone analysis. After this phase, users can generate newsletter drafts from inputs (Slack threads, meeting notes, raw text), get AI-suggested subject lines, and receive readability feedback before sending.

### Tasks

#### 7.1 — AI Content Generation Service

**What**: Build the AI content generation service that drafts newsletter content from disparate input sources using the Anthropic Claude API with prompt caching.

**Design**:

```typescript
// packages/api/src/services/ai-content-generator.ts
import Anthropic from "@anthropic-ai/sdk";

const anthropic = new Anthropic();

export interface ContentInput {
  type: "raw_text" | "slack_messages" | "meeting_notes" | "bullet_points";
  content: string;
  metadata?: { channel?: string; date?: string; participants?: string[] };
}

export interface GenerationOptions {
  tone: "professional" | "casual" | "executive";
  maxWords?: number;
  targetAudience?: string;     // e.g., "Engineering team", "All employees"
  brandVoiceGuidelines?: string;
  includeCallToAction?: boolean;
}

export interface GeneratedContent {
  blocks: Array<{
    type: "heading" | "text" | "button";
    content: string;
    mjml: string;
  }>;
  subjectSuggestions: Array<{
    text: string;
    rationale: string;
  }>;
  readabilityScore: number;      // 0-100 (Flesch-Kincaid adapted)
  estimatedReadTime: number;     // seconds
  tokensUsed: number;
}

export async function generateNewsletterContent(
  inputs: ContentInput[],
  options: GenerationOptions,
  organisationContext: { name: string; branding: object }
): Promise<GeneratedContent> {
  const systemPrompt = `You are a professional internal communications writer for ${organisationContext.name}.
Your task is to draft an internal newsletter from the provided source material.

Guidelines:
- Tone: ${options.tone}
- Target audience: ${options.targetAudience ?? "all employees"}
- Maximum length: ${options.maxWords ?? 500} words
${options.brandVoiceGuidelines ? `- Brand voice: ${options.brandVoiceGuidelines}` : ""}

Structure your output as a JSON object with:
1. "blocks": array of content blocks, each with "type" (heading/text/button), "content" (plain text), and "mjml" (MJML markup)
2. "subjectSuggestions": array of 3 subject line suggestions, each with "text" and "rationale"
3. "readabilityScore": Flesch-Kincaid readability score (0-100, higher = easier to read)
4. "estimatedReadTime": estimated read time in seconds

Keep sections brief and scannable. Use bullet points for lists. Lead with the most important news.`;

  const userContent = inputs.map((input) => {
    return `--- ${input.type.toUpperCase()} ---\n${input.content}`;
  }).join("\n\n");

  const response = await anthropic.messages.create({
    model: "claude-sonnet-4-20250514",
    max_tokens: 4096,
    system: [{ type: "text", text: systemPrompt, cache_control: { type: "ephemeral" } }],
    messages: [{ role: "user", content: userContent }],
  });

  // Parse the structured JSON response
  const parsed = JSON.parse(response.content[0].text);
  return { ...parsed, tokensUsed: response.usage.input_tokens + response.usage.output_tokens };
}
```

API endpoints:
```typescript
// POST /api/ai/generate
interface GenerateRequest {
  inputs: ContentInput[];
  options: GenerationOptions;
  campaignId?: string;       // If provided, saves generated content to the campaign
}
interface GenerateResponse extends GeneratedContent {}

// POST /api/ai/suggest-subject
interface SuggestSubjectRequest {
  content: string;           // Newsletter content (plain text or HTML)
  previousSubjects?: string[]; // Past subjects to avoid repetition
  count?: number;            // Number of suggestions (default: 3)
}
interface SuggestSubjectResponse {
  suggestions: Array<{ text: string; rationale: string }>;
}

// POST /api/ai/rewrite
interface RewriteRequest {
  content: string;
  instruction: string;   // e.g., "make it shorter", "more formal tone", "add bullet points"
}
interface RewriteResponse {
  content: string;
  mjml: string;
  tokensUsed: number;
}
```

**Testing**:
- `Integration (mocked Claude): generateNewsletterContent returns structured blocks`
- `Integration (mocked Claude): generated content includes 3 subject suggestions`
- `Integration (mocked Claude): readabilityScore is between 0 and 100`
- `Unit: system prompt includes organisation name and tone`
- `Unit: system prompt uses prompt caching (cache_control set on system message)`
- `Integration: POST /api/ai/generate with campaignId saves blocks to campaign`
- `Integration: POST /api/ai/generate without ANTHROPIC_API_KEY returns 503`
- `Integration: POST /api/ai/suggest-subject returns subject suggestions`
- `Integration: POST /api/ai/rewrite returns rewritten content`
- `Unit: ContentInput schema validates type enum`

---

#### 7.2 — Readability Scoring and Writing Coach

**What**: Implement real-time readability analysis that scores newsletter content and provides specific improvement suggestions.

**Design**:

```typescript
// packages/api/src/services/readability-scorer.ts
export interface ReadabilityAnalysis {
  overallScore: number;           // 0-100
  fleschKincaidGrade: number;     // Grade level
  avgSentenceLength: number;      // Words per sentence
  avgWordLength: number;          // Syllables per word
  longSentences: Array<{ text: string; wordCount: number; suggestion: string }>;
  passiveVoice: Array<{ text: string; suggestion: string }>;
  jargon: Array<{ word: string; suggestion: string }>;
  estimatedReadTime: number;      // Seconds
}

export function analyzeReadability(text: string): ReadabilityAnalysis {
  // 1. Strip HTML tags
  // 2. Split into sentences
  // 3. Calculate Flesch-Kincaid grade level
  // 4. Identify sentences > 25 words (flag as long)
  // 5. Detect passive voice patterns
  // 6. Check for corporate jargon ("synergy", "leverage", "circle back")
  // 7. Calculate estimated read time (238 words/minute average)
  return analysis;
}

// POST /api/ai/analyze-readability
interface AnalyzeReadabilityRequest {
  content: string;    // Plain text or HTML
}
interface AnalyzeReadabilityResponse extends ReadabilityAnalysis {}
```

Jargon dictionary:
```typescript
const CORPORATE_JARGON: Record<string, string> = {
  "synergy": "collaboration",
  "leverage": "use",
  "circle back": "follow up",
  "deep dive": "detailed look",
  "bandwidth": "capacity",
  "move the needle": "make progress",
  "low-hanging fruit": "easy wins",
  "paradigm shift": "major change",
  "stakeholder": "team member/partner",
  "ideation": "brainstorming",
};
```

**Testing**:
- `Unit: analyzeReadability returns score between 0 and 100`
- `Unit: sentence with 30 words is flagged as long`
- `Unit: "The report was written by Jane" is flagged as passive voice`
- `Unit: "leverage our synergies" flags both words as jargon`
- `Unit: estimated read time for 238 words is approximately 60 seconds`
- `Unit: Flesch-Kincaid grade for simple text (Grade 6) is calculated correctly`
- `Integration: POST /api/ai/analyze-readability returns ReadabilityAnalysis`
- `Fixture-based: analyze readability of a real newsletter sample (fixture)`

---

## Phase 8: Dashboard Frontend

### Purpose
Build the Next.js web dashboard that provides the user interface for all backend capabilities: campaign management, template editing, subscriber management, analytics visualisation, and AI content generation. After this phase, the application is fully usable through a web browser.

### Tasks

#### 8.1 — Dashboard Layout and Navigation

**What**: Build the authenticated dashboard shell with sidebar navigation, organisation switcher, and responsive layout.

**Design**:

```typescript
// packages/web/src/app/(dashboard)/layout.tsx
// Sidebar navigation items:
const navItems = [
  { label: "Campaigns", href: "/campaigns", icon: "Mail" },
  { label: "Templates", href: "/templates", icon: "Layout" },
  { label: "Subscribers", href: "/subscribers", icon: "Users" },
  { label: "Lists", href: "/lists", icon: "ListFilter" },
  { label: "Analytics", href: "/analytics", icon: "BarChart3" },
  { label: "Media", href: "/media", icon: "Image" },
  { label: "Settings", href: "/settings", icon: "Settings" },
];

// Layout structure:
// ┌────────────────────────────────────────────────┐
// │ TopBar: org name, user menu, notifications     │
// ├──────────┬─────────────────────────────────────┤
// │ Sidebar  │                                     │
// │          │  Page Content                       │
// │ nav      │  (children)                         │
// │ items    │                                     │
// │          │                                     │
// └──────────┴─────────────────────────────────────┘
```

API client:
```typescript
// packages/web/src/lib/api-client.ts
const API_BASE = process.env.NEXT_PUBLIC_API_URL ?? "http://localhost:3001";

export async function apiClient<T>(
  path: string,
  options: RequestInit = {}
): Promise<T> {
  const response = await fetch(`${API_BASE}${path}`, {
    ...options,
    credentials: "include",
    headers: { "Content-Type": "application/json", ...options.headers },
  });
  if (!response.ok) {
    const error = await response.json().catch(() => ({}));
    throw new ApiError(response.status, error.message ?? "Unknown error");
  }
  return response.json();
}
```

**Testing**:
- `E2E (Playwright): authenticated user sees dashboard with sidebar navigation`
- `E2E (Playwright): clicking each nav item navigates to the correct page`
- `E2E (Playwright): unauthenticated user is redirected to login page`
- `E2E (Playwright): sidebar collapses on mobile viewport`
- `Unit: apiClient appends credentials and Content-Type header`
- `Unit: apiClient throws ApiError on non-200 response`

---

#### 8.2 — Campaign Management Pages

**What**: Build the campaign list, creation wizard, content editor, and send/schedule flow pages.

**Design**:

Pages:
- `/campaigns` — list view with filters (status, date range), search, and pagination
- `/campaigns/new` — creation wizard: name, subject, template selection, audience selection
- `/campaigns/[id]` — campaign detail/editor with tabs: Content, Audience, Preview, Schedule
- `/campaigns/[id]/analytics` — campaign analytics (links to Phase 8.4)

Campaign creation wizard flow:
```
Step 1: Basic Info (name, subject, preview text, from/reply-to)
    ↓
Step 2: Select Template (grid of template thumbnails)
    ↓
Step 3: Edit Content (GrapesJS editor with template blocks loaded)
    ↓
Step 4: Select Audience (list picker with subscriber count preview)
    ↓
Step 5: Review & Send/Schedule (preview, accessibility check, send options)
```

**Testing**:
- `E2E (Playwright): campaign list displays campaigns with correct status badges`
- `E2E (Playwright): create campaign wizard completes all 5 steps`
- `E2E (Playwright): template selection loads template thumbnails`
- `E2E (Playwright): content editor loads GrapesJS with template blocks`
- `E2E (Playwright): audience step shows subscriber count for selected lists`
- `E2E (Playwright): schedule sets future date/time and shows confirmation`
- `E2E (Playwright): send now shows confirmation dialog before sending`
- `E2E (Playwright): campaign with "sent" status cannot be edited`

---

#### 8.3 — Subscriber and List Management Pages

**What**: Build the subscriber list, detail view, CSV import wizard, and mailing list management pages.

**Design**:

Pages:
- `/subscribers` — paginated list with search, department filter, status filter
- `/subscribers/[id]` — subscriber detail: profile, preferences, engagement history
- `/subscribers/import` — CSV import wizard with column mapping and preview
- `/lists` — list management with manual/dynamic badge
- `/lists/new` — create list wizard (manual: pick subscribers; dynamic: build filter)
- `/lists/[id]` — list detail with subscriber preview

CSV import wizard:
```
Step 1: Upload CSV file
    ↓
Step 2: Column mapping (auto-detect + manual correction)
    ↓
Step 3: Preview first 10 rows with validation status
    ↓
Step 4: Import results (imported, skipped, errors)
```

Dynamic list filter builder UI:
```typescript
interface FilterBuilderProps {
  value: ListFilter;
  onChange: (filter: ListFilter) => void;
  availableFields: Array<{ key: string; label: string; type: "string" | "number" | "boolean" }>;
}
// Renders: [Field dropdown] [Operator dropdown] [Value input] [+ Add condition]
// Supports AND/OR toggle
```

**Testing**:
- `E2E (Playwright): subscriber list displays with pagination controls`
- `E2E (Playwright): search by name filters subscriber list`
- `E2E (Playwright): CSV import wizard processes fixture CSV file`
- `E2E (Playwright): CSV import shows validation errors for invalid rows`
- `E2E (Playwright): dynamic list filter builder adds conditions and previews count`
- `E2E (Playwright): subscriber detail shows engagement history timeline`

---

#### 8.4 — Analytics Dashboard Pages

**What**: Build the analytics dashboard with campaign performance charts, engagement heatmaps, department comparisons, and organisation-level trends.

**Design**:

Pages:
- `/analytics` — organisation overview with trend charts and top campaigns
- `/campaigns/[id]/analytics` — single campaign analytics with:
  - Summary cards (sent, delivered, opens, clicks, bounces)
  - Open/click rate gauge charts
  - Department comparison bar chart
  - Device/client pie charts
  - Link click ranking table
  - Hourly engagement heatmap
  - Subscriber-level engagement table (sortable, searchable)

Chart library: Recharts (React-native, composable, responsive)

```typescript
// packages/web/src/components/analytics/CampaignSummaryCards.tsx
interface SummaryCardProps {
  label: string;
  value: number;
  percentage?: number;
  trend?: "up" | "down" | "flat";
}

// packages/web/src/components/analytics/DepartmentChart.tsx
// Horizontal bar chart comparing open rates across departments

// packages/web/src/components/analytics/EngagementHeatmap.tsx
// Grid showing engagement intensity by hour of day (x) and day of week (y)
```

**Testing**:
- `E2E (Playwright): analytics overview loads trend charts`
- `E2E (Playwright): campaign analytics shows summary cards with correct values`
- `E2E (Playwright): department chart renders bars for each department`
- `E2E (Playwright): subscriber engagement table is sortable by open count`
- `E2E (Playwright): link click ranking shows tracked links ordered by clicks`
- `Unit: SummaryCard renders label, value, and percentage`
- `Unit: DepartmentChart handles zero-division (department with 0 sent)`

---

#### 8.5 — Settings Pages

**What**: Build the organisation settings pages: general settings, sending domains, integrations, team management, and branding.

**Design**:

Pages:
- `/settings` — general: organisation name, default from/reply-to, timezone
- `/settings/domains` — sending domain list with verification status badges
- `/settings/domains/new` — domain setup wizard with DNS record instructions
- `/settings/team` — team member list, invite form, role management
- `/settings/branding` — logo upload, primary colour picker
- `/settings/integrations` — HRIS/SCIM configuration, OAuth connections

**Testing**:
- `E2E (Playwright): settings page saves organisation name change`
- `E2E (Playwright): domain setup wizard displays required DNS records`
- `E2E (Playwright): verify domain button triggers DNS check and shows results`
- `E2E (Playwright): team page lists members with role badges`
- `E2E (Playwright): invite form sends invitation and adds pending member`
- `E2E (Playwright): branding page uploads logo and updates preview`

---

## Phase 9: Send-Time Optimisation & Smart Segmentation

### Purpose
Add ML-powered engagement features: per-subscriber send-time optimisation that predicts the best time to deliver each email, and smart audience segmentation that clusters subscribers by engagement behaviour. These are the AI-native differentiators that separate this platform from basic newsletter tools.

### Tasks

#### 9.1 — Engagement Profile Computation

**What**: Build the background job that analyses each subscriber's historical engagement events and updates their engagement_profile JSONB with ML-derived metrics.

**Design**:

```typescript
// packages/api/src/services/engagement-profiler.ts
export interface ComputedEngagementProfile {
  engagementScore: number;        // 0.0 - 1.0 (weighted: opens * 0.4 + clicks * 0.6)
  segment: "highly_engaged" | "moderately_engaged" | "low_engagement" | "disengaged";
  avgOpenRate: number;
  avgClickRate: number;
  bestSendHourUtc: number;        // 0-23, hour with highest open probability
  bestSendDay: number;            // 0-6, day with highest open probability
  preferredDevice: string;
  preferredClient: string;
  lastOpenedAt: string | null;
  topicsOfInterest: string[];     // Derived from clicked content categories
  churnRisk: number;              // 0.0 - 1.0 (high = likely to disengage)
}

export async function computeEngagementProfile(
  subscriberId: string,
  lookbackDays: number = 90
): Promise<ComputedEngagementProfile> {
  // 1. Fetch all engagement events for subscriber in lookback window
  // 2. Calculate open rate and click rate across campaigns
  // 3. Find most common open hour (UTC) and day of week
  // 4. Find most common device type and email client
  // 5. Calculate engagement score (weighted composite)
  // 6. Assign segment based on score thresholds
  // 7. Calculate churn risk (days since last open / lookback window)
  // 8. Extract topics from clicked link categories
}

// BullMQ repeatable job: runs nightly at 02:00 UTC
export const engagementProfileQueue = new Queue("engagement-profile", { connection: redisConnection });
engagementProfileQueue.add("compute-all", {}, {
  repeat: { pattern: "0 2 * * *" },
});
```

**Testing**:
- `Unit: engagement score with 80% opens and 40% clicks = 0.56 (0.8*0.4 + 0.4*0.6)`
- `Unit: segment assignment: score >= 0.6 = "highly_engaged"`
- `Unit: segment assignment: score < 0.1 = "disengaged"`
- `Unit: bestSendHourUtc is the mode of open event hours`
- `Unit: churnRisk with last open 45 days ago (of 90 day window) = 0.5`
- `Integration: computeEngagementProfile updates subscriber.engagement_profile JSONB`
- `Integration: nightly job processes all active subscribers`

---

#### 9.2 — Send-Time Optimisation

**What**: Implement per-subscriber send-time optimisation that staggers campaign delivery so each subscriber receives the email at their predicted optimal time.

**Design**:

```typescript
// packages/api/src/services/send-time-optimiser.ts
export interface OptimisedSendSchedule {
  subscriberId: string;
  scheduledSendAt: Date;
  confidence: number;
}

export function computeOptimisedSchedule(
  subscribers: Array<{ id: string; engagementProfile: EngagementProfile; timezone: string | null }>,
  campaignScheduledAt: Date,
  spreadWindowHours: number = 4  // Max spread: deliver within 4 hours of scheduled time
): OptimisedSendSchedule[] {
  return subscribers.map((sub) => {
    const profile = sub.engagementProfile;
    if (!profile.bestSendHourUtc || profile.engagementScore === undefined) {
      // No data: send at the campaign's scheduled time
      return { subscriberId: sub.id, scheduledSendAt: campaignScheduledAt, confidence: 0 };
    }

    // Calculate the best send time in UTC based on subscriber's historical patterns
    const optimalHour = profile.bestSendHourUtc;
    const optimalDay = profile.bestSendDay;

    // Constrain to spread window around campaign scheduled time
    const scheduledSendAt = constrainToWindow(
      campaignScheduledAt,
      optimalHour,
      spreadWindowHours
    );

    return {
      subscriberId: sub.id,
      scheduledSendAt,
      confidence: profile.engagementScore,
    };
  });
}
```

Campaign send flow with STO:
```
1. User schedules campaign at 09:00 UTC
2. System computes optimised schedule for each subscriber
3. Subscribers are grouped by send time (15-minute buckets)
4. BullMQ delayed jobs are created for each bucket
5. Each bucket sends to its subscribers at the optimised time
6. Subscribers without engagement data receive at the original 09:00 time
```

**Testing**:
- `Unit: subscriber with bestSendHourUtc=14 gets scheduled at 14:00 UTC`
- `Unit: subscriber with no engagement data gets the campaign's scheduled time`
- `Unit: all optimised times fall within the spread window`
- `Unit: subscribers are grouped into 15-minute buckets`
- `Integration: campaign with STO enabled creates delayed BullMQ jobs`
- `Integration: each bucket sends to the correct subscribers`

---

#### 9.3 — Smart Audience Segmentation

**What**: Build ML-powered audience segments that automatically cluster subscribers by engagement patterns, enabling targeted content without manual list management.

**Design**:

```typescript
// POST /api/ai/smart-segments/generate
// Analyses subscriber engagement profiles and creates smart segments
interface GenerateSegmentsRequest {
  minSegmentSize: number;    // Minimum subscribers per segment (default: 10)
  maxSegments: number;       // Maximum segments to create (default: 5)
}
interface GenerateSegmentsResponse {
  segments: Array<{
    name: string;            // AI-generated name: "Active Readers", "At-Risk", etc.
    description: string;
    subscriberCount: number;
    avgEngagementScore: number;
    criteria: string;        // Human-readable description of what defines this segment
  }>;
}

// GET /api/ai/smart-segments
// List existing smart segments

// GET /api/ai/smart-segments/:id/subscribers
// List subscribers in a smart segment
```

Segmentation uses k-means clustering on engagement features:
```typescript
export function clusterSubscribers(
  profiles: Array<{ subscriberId: string; engagementScore: number; avgOpenRate: number; avgClickRate: number; churnRisk: number }>,
  k: number
): Map<number, string[]> {
  // k-means clustering on [engagementScore, avgOpenRate, avgClickRate, churnRisk]
  // Returns: cluster_id -> subscriber_ids
}
```

**Testing**:
- `Unit: clusterSubscribers with 100 subscribers and k=3 returns 3 clusters`
- `Unit: each cluster has at least minSegmentSize subscribers`
- `Unit: highly engaged subscribers cluster together (avg score > 0.6)`
- `Integration: POST /api/ai/smart-segments/generate creates smart segments`
- `Integration: generated segments have AI-generated names and descriptions`
- `Integration: GET /api/ai/smart-segments/:id/subscribers returns cluster members`

---

## Phase 10: Multichannel Delivery (SMS & Push)

### Purpose
Extend delivery beyond email to include SMS and push notification channels, allowing newsletters to reach frontline employees who may not check email regularly. Content is auto-formatted per channel: full HTML for email, condensed text for SMS, and headline+excerpt for push.

### Tasks

#### 10.1 — Delivery Channel Abstraction

**What**: Refactor the delivery provider interface to support multiple channels, and implement SMS delivery via Twilio and web push via Web Push API.

**Design**:

```typescript
// packages/api/src/services/delivery-provider.ts
export type DeliveryChannel = "email" | "sms" | "push";

export interface ChannelMessage {
  channel: DeliveryChannel;
  to: string;              // Email address, phone number, or push subscription
  content: {
    subject?: string;       // Email/push only
    html?: string;          // Email only
    text: string;           // All channels (SMS body, push body, email plaintext)
    excerpt?: string;       // Push notification body
  };
  metadata?: Record<string, unknown>;
}

export interface ChannelProvider {
  channel: DeliveryChannel;
  send(message: ChannelMessage): Promise<DeliveryResult>;
}

// SMS provider (Twilio)
export class TwilioSmsProvider implements ChannelProvider {
  channel = "sms" as const;
  async send(message: ChannelMessage): Promise<DeliveryResult> {
    // Twilio API: POST /Messages
    // Truncate text to 1600 chars (SMS concatenation limit)
    // Include opt-out text: "Reply STOP to unsubscribe"
  }
}

// Web Push provider
export class WebPushProvider implements ChannelProvider {
  channel = "push" as const;
  async send(message: ChannelMessage): Promise<DeliveryResult> {
    // VAPID-authenticated Web Push API
    // Payload: { title: subject, body: excerpt, icon: orgLogo, url: webViewUrl }
  }
}
```

Campaign now supports channel selection:
```typescript
// PATCH /api/campaigns/:id
interface UpdateCampaignRequest {
  channels?: DeliveryChannel[];  // ["email", "sms", "push"]
}
```

**Testing**:
- `Unit: TwilioSmsProvider truncates message to 1600 characters`
- `Unit: TwilioSmsProvider appends opt-out text`
- `Unit: WebPushProvider constructs correct VAPID push payload`
- `Integration (mocked Twilio): SMS send returns messageId`
- `Integration (mocked push): push notification delivered successfully`
- `Integration: campaign with channels=["email","sms"] sends both channels`
- `Unit: content auto-formatter converts HTML email to plain text for SMS`

---

#### 10.2 — Subscriber Channel Preferences

**What**: Allow subscribers to choose their preferred delivery channels and manage phone number and push subscription registration.

**Design**:

```typescript
// PATCH /api/subscribers/:id/preferences
interface UpdatePreferencesRequest {
  channels?: {
    email: boolean;
    sms: boolean;
    push: boolean;
  };
  phoneNumber?: string;     // E.164 format for SMS
}

// POST /api/subscribers/:id/push-subscription
// Register a Web Push subscription (from browser)
interface PushSubscriptionRequest {
  endpoint: string;
  keys: { p256dh: string; auth: string };
}
```

Campaign send worker resolves channels per subscriber:
```typescript
// For each subscriber:
// 1. Check subscriber.preferences.channels
// 2. For each enabled channel that the campaign also targets:
//    a. Format content for that channel
//    b. Send via the channel provider
//    c. Record engagement_event with channel metadata
```

**Testing**:
- `Integration: subscriber with sms=true and email=false only receives SMS`
- `Integration: subscriber with no channel preferences defaults to email`
- `Integration: push subscription stores endpoint and keys`
- `Unit: phone number validation accepts E.164 format (+1234567890)`
- `Unit: phone number validation rejects non-E.164 format`

---

## Phase 11: Advanced Features & Polish

### Purpose
Implement remaining differentiating features: sentiment analysis via embedded surveys, longitudinal engagement tracking, template version history, campaign cloning, and the public API with API key authentication. After this phase, the platform covers all MVP and v1.1 features from the feature specification.

### Tasks

#### 11.1 — Embedded Surveys and Sentiment Tracking

**What**: Allow newsletter authors to embed 1-click surveys (thumbs up/down, 1-5 stars, emoji reaction) in newsletters and aggregate sentiment data.

**Design**:

```typescript
// Survey block type added to TemplateBlockSchema
// type: "survey"
// mjml: rendered as clickable icons (thumbs up/down, stars, emojis)
// Each option links to: /api/surveys/respond?campaignId=X&subscriberId=Y&response=Z

// POST /api/surveys/respond (no auth required — token-based)
interface SurveyResponseRequest {
  token: string;           // JWT: { campaignId, subscriberId, blockId }
  response: string;        // "thumbs_up", "thumbs_down", "1"-"5", emoji
}

// GET /api/campaigns/:id/survey-results
interface SurveyResultsResponse {
  blockId: string;
  totalResponses: number;
  breakdown: Record<string, number>;  // { "thumbs_up": 85, "thumbs_down": 12 }
  responseRate: number;
}
```

**Testing**:
- `Integration: survey response records engagement_event with event_type="survey_response"`
- `Integration: GET survey results returns correct breakdown`
- `Integration: duplicate response from same subscriber updates (not duplicates)`
- `Unit: survey token validates campaignId and subscriberId`

---

#### 11.2 — Public REST API with API Key Authentication

**What**: Expose a public API for external integrations, authenticated via organisation-scoped API keys.

**Design**:

```typescript
// POST /api/settings/api-keys
interface CreateApiKeyRequest {
  name: string;
  scopes: Array<"subscribers:read" | "subscribers:write" | "campaigns:read" | "campaigns:write" | "analytics:read">;
}
// Returns: { id, name, key: "nlb_live_xxx..." } (key shown only once)

// Public API endpoints (under /api/v1/):
// All authenticated via X-Api-Key header
// GET    /api/v1/subscribers
// POST   /api/v1/subscribers
// PATCH  /api/v1/subscribers/:id
// GET    /api/v1/campaigns
// POST   /api/v1/campaigns
// GET    /api/v1/campaigns/:id/analytics
// GET    /api/v1/lists
```

API key format: `nlb_live_{32 random hex chars}`. Stored as Argon2id hash. Matched via prefix index.

**Testing**:
- `Integration: API key creation returns key with nlb_live_ prefix`
- `Integration: GET /api/v1/subscribers with valid API key returns subscribers`
- `Integration: GET /api/v1/subscribers without API key returns 401`
- `Integration: API key with subscribers:read scope cannot POST subscribers`
- `Unit: API key hash verification succeeds with correct key`

---

#### 11.3 — Campaign Cloning and Template Version History

**What**: Allow cloning existing campaigns and maintaining template version history for rollback.

**Design**:

```typescript
// POST /api/campaigns/:id/clone
// Creates a new draft campaign with the same content, subject, and audience
// Appends "(copy)" to the name
// Returns: new campaign object

// GET /api/templates/:id/versions
// Returns version history (version number, updatedAt, updatedBy)

// POST /api/templates/:id/versions/:version/restore
// Restores template blocks and settings from a previous version
```

Template versioning: each save increments `template.version` and stores the previous blocks/settings in a `template_version` table.

```typescript
export const templateVersion = pgTable("template_version", {
  id: uuid("id").primaryKey().defaultRandom(),
  templateId: uuid("template_id").notNull().references(() => template.id, { onDelete: "cascade" }),
  version: integer("version").notNull(),
  blocks: jsonb("blocks").notNull(),
  settings: jsonb("settings").notNull(),
  createdBy: uuid("created_by").references(() => appUser.id),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});
```

**Testing**:
- `Integration: POST /api/campaigns/:id/clone creates draft copy with "(copy)" name`
- `Integration: cloned campaign has independent content blocks (editing one does not affect the other)`
- `Integration: template save creates version history entry`
- `Integration: GET /api/templates/:id/versions returns ordered version list`
- `Integration: POST /api/templates/:id/versions/:version/restore reverts to previous blocks`

---

## Phase 12: Production Hardening & Deployment

### Purpose
Prepare for production deployment: rate limiting, error monitoring, structured logging, database connection pooling, health checks, backup strategies, and production Docker configuration. After this phase, the application is ready for self-hosted production deployment.

### Tasks

#### 12.1 — Rate Limiting and Security Hardening

**What**: Add rate limiting to all API endpoints, CSRF protection, and security headers.

**Design**:

```typescript
// packages/api/src/middleware/rate-limit.ts
import { fastifyRateLimit } from "@fastify/rate-limit";

// Global rate limit: 100 requests per minute per IP
await app.register(fastifyRateLimit, {
  max: 100,
  timeWindow: "1 minute",
  keyGenerator: (request) => request.ip,
});

// Per-route overrides:
// POST /api/auth/login: 10 per minute (brute force protection)
// POST /api/ai/*: 20 per minute (AI cost control)
// POST /api/webhooks/*: 1000 per minute (high-volume webhook events)
// GET /t/*: 5000 per minute (link tracking — high volume)
```

Security headers:
```typescript
import helmet from "@fastify/helmet";
await app.register(helmet, {
  contentSecurityPolicy: false,  // Handled by Next.js for frontend
  crossOriginEmbedderPolicy: false,
});
```

**Testing**:
- `Integration: 101st request within 1 minute returns 429`
- `Integration: 11th login attempt within 1 minute returns 429`
- `Integration: X-RateLimit-Remaining header decrements correctly`
- `Unit: security headers include X-Frame-Options and X-Content-Type-Options`

---

#### 12.2 — Structured Logging and Error Monitoring

**What**: Implement structured JSON logging with request correlation IDs and error boundary middleware.

**Design**:

```typescript
// packages/api/src/plugins/logger.ts
import pino from "pino";

export const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  serializers: {
    req: (req) => ({
      method: req.method,
      url: req.url,
      organisationId: req.organisationId,
      userId: req.userId,
      correlationId: req.headers["x-correlation-id"],
    }),
    err: pino.stdSerializers.err,
  },
});

// Error boundary middleware
app.setErrorHandler((error, request, reply) => {
  logger.error({ err: error, req: request }, "Unhandled error");

  if (error.validation) {
    return reply.status(400).send({ error: "Validation Error", details: error.validation });
  }

  reply.status(500).send({ error: "Internal Server Error" });
});
```

**Testing**:
- `Unit: logger serializes request with organisationId and correlationId`
- `Integration: validation error returns 400 with details`
- `Integration: unhandled error returns 500 and logs error with stack trace`
- `Integration: each request log entry includes correlation ID`

---

#### 12.3 — Production Docker Configuration

**What**: Optimise the Docker configuration for production: multi-stage build, non-root user, health checks, and environment variable documentation.

**Design**:

Updated `Dockerfile` for production:
```dockerfile
FROM node:22-alpine AS base
RUN corepack enable && corepack prepare pnpm@latest --activate
RUN addgroup --system --gid 1001 nodejs && adduser --system --uid 1001 newsletter

FROM base AS deps
WORKDIR /app
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY packages/shared/package.json packages/shared/
COPY packages/api/package.json packages/api/
COPY packages/web/package.json packages/web/
RUN pnpm install --frozen-lockfile --prod

FROM base AS builder
WORKDIR /app
COPY --from=deps /app ./
COPY . .
RUN pnpm build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV=production
USER newsletter
COPY --from=builder --chown=newsletter:nodejs /app/packages/api/dist ./packages/api/dist
COPY --from=builder --chown=newsletter:nodejs /app/packages/shared/dist ./packages/shared/dist
COPY --from=builder --chown=newsletter:nodejs /app/packages/web/.next ./packages/web/.next
COPY --from=deps --chown=newsletter:nodejs /app/node_modules ./node_modules

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -q --spider http://localhost:3001/health || exit 1

EXPOSE 3001
CMD ["node", "packages/api/dist/server.js"]
```

Production `docker-compose.prod.yml`:
```yaml
version: "3.8"
services:
  app:
    build: .
    ports:
      - "3001:3001"
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - SESSION_SECRET=${SESSION_SECRET}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    volumes:
      - redisdata:/data
    command: redis-server --save 60 1 --loglevel warning
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  pgdata:
  redisdata:
```

**Testing**:
- `Integration: docker build completes successfully`
- `Integration: docker image runs as non-root user (uid 1001)`
- `Integration: health check endpoint responds within 5 seconds`
- `Integration: docker compose up -d starts all services with health checks passing`
- `Integration: application recovers after PostgreSQL restart (connection pool reconnection)`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Project Scaffolding          ─── required by everything
    │
Phase 2: Authentication & Multi-Tenancy            ─── requires Phase 1
    │
    ├── Phase 3: Subscriber Management & HRIS       ─── requires Phase 2
    │       │
    │       └── Phase 4: Template System & Rendering ─── requires Phase 2, benefits from Phase 3
    │               │
    │               └── Phase 5: Campaign Builder    ─── requires Phases 3 + 4
    │                       │
    │                       ├── Phase 6: Analytics   ─── requires Phase 5
    │                       │       │
    │                       │       └── Phase 9: Send-Time Optimisation ─── requires Phase 6
    │                       │
    │                       ├── Phase 7: AI Content  ─── requires Phase 5, can parallel with Phase 6
    │                       │
    │                       └── Phase 8: Dashboard   ─── requires Phases 3-7 (incremental)
    │
    ├── Phase 10: Multichannel Delivery             ─── requires Phase 5
    │
    ├── Phase 11: Advanced Features                 ─── requires Phases 5-8
    │
    └── Phase 12: Production Hardening              ─── can start after Phase 5, applies to all
```

### Parallelism Opportunities

- **Phases 6 and 7** can be developed concurrently after Phase 5 (analytics and AI content generation are independent)
- **Phase 8** (Dashboard) can be developed incrementally alongside Phases 3-7, building pages as backend APIs become available
- **Phase 10** (Multichannel) can start after Phase 5 and proceed in parallel with Phases 6, 7, 9
- **Phase 12** (Production Hardening) can start after Phase 5 with incremental additions as new features land

---

## Definition of Done (per phase)

1. All tasks implemented with working code.
2. All unit and integration tests pass (`pnpm test` exits with code 0).
3. ESLint passes with zero errors (`pnpm lint`).
4. TypeScript strict mode compiles with zero errors (`pnpm build`).
5. Docker build succeeds and health check passes.
6. Database migrations generated and applied successfully.
7. Feature works end-to-end (manual verification or E2E test).
8. New API endpoints appear in OpenAPI spec (`/docs/json`).
9. New environment variables documented in `.env.example`.
10. Zod schemas validate all JSONB field shapes at API boundaries.
11. No secrets or credentials committed to version control.
12. Accessibility checker passes on any new email template output.
