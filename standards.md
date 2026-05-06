# Standards & API Reference

> Project: Internal Newsletter Builder · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 27001:2022** — Information security management; governs access controls for internal newsletter platforms managing sensitive employee communications, HR announcements, and confidential company updates; Annex A A.8.3 (Information access restriction) and A.5.14 (Information transfer). URL: https://www.iso.org/standard/82875.html

- **ISO/IEC 27018:2019 — Protection of PII in Public Clouds** — Governs processing of employee personal data (email addresses, engagement analytics, click tracking) in cloud-hosted internal newsletter platforms; requires data minimisation and retention policy transparency. URL: https://www.iso.org/standard/76559.html

### W3C & IETF Standards

- **RFC 5321 — SMTP: Simple Mail Transfer Protocol** — Foundational email transmission standard; internal newsletter platforms deliver employee newsletters through SMTP-based email infrastructure (SendGrid, Amazon SES, Postmark, or self-hosted mail servers). URL: https://datatracker.ietf.org/doc/html/rfc5321

- **RFC 5322 — Internet Message Format** — Defines the syntax of email message headers and body; governs the structure of newsletter emails including subject line, From/Reply-To headers, and MIME multipart body (HTML + plain text alternative). URL: https://datatracker.ietf.org/doc/html/rfc5322

- **RFC 2045-2049 — MIME: Multipurpose Internet Mail Extensions** — Defines multipart email format with HTML and plain text parts, inline images (cid: references), and file attachments used in newsletter delivery. URL: https://datatracker.ietf.org/doc/html/rfc2045

- **RFC 7208 — SPF: Sender Policy Framework** — Required DNS authentication for all sending domains; mandatory for deliverability to Gmail (enforced February 2024), Outlook (enforced May 2025), and Yahoo; internal newsletter platforms must configure SPF for all custom sending domains. URL: https://datatracker.ietf.org/doc/html/rfc7208

- **RFC 6376 — DKIM: DomainKeys Identified Mail** — Required cryptographic email signing for deliverability; mandated by Gmail and Outlook for bulk senders; internal newsletter platforms must sign outbound email with DKIM for each sending domain. URL: https://datatracker.ietf.org/doc/html/rfc6376

- **RFC 7489 — DMARC** — Domain-level email authentication policy required for bulk email sending; prevents spoofing of internal newsletter sending domains; DMARCbis progressing as IETF Proposed Standard in 2025. URL: https://datatracker.ietf.org/doc/html/rfc7489

- **RFC 8058 — One-Click List-Unsubscribe** — Defines the List-Unsubscribe-Post header enabling one-click unsubscribe; required by Gmail and Yahoo for bulk senders; applies to internal newsletters with employee opt-out rights. URL: https://datatracker.ietf.org/doc/html/rfc8058

- **RFC 6749 — OAuth 2.0** — Authorization framework used by internal newsletter platforms for HR system integrations (Workday, BambooHR, HRIS) and single sign-on connections with enterprise identity providers. URL: https://datatracker.ietf.org/doc/html/rfc6749

- **W3C WCAG 2.2 — Web Content Accessibility Guidelines** — Governs accessibility of HTML email content; internal newsletters must meet accessibility standards (alt text for images, sufficient colour contrast, logical heading structure) for employees with disabilities; Section 508 compliance required for US federal agencies. URL: https://www.w3.org/TR/WCAG22/

### Data Model & API Specifications

- **MJML — Responsive Email Framework** — MIT-licensed markup language (created by Mailjet) designed to simplify responsive email HTML development; compiles to email-client-compatible HTML; handles Outlook quirks automatically; the de facto standard for programmatic email template authoring. URL: https://mjml.io/

- **React Email** — Open-source (MIT) React component library for building email templates with JSX; alternative to MJML for React-native development workflows; type-safe email template authoring with hot-reload preview. URL: https://react.email/

- **mjml-react (Wix Incubator)** — Open-source library providing React components that wrap MJML elements; generates MJML/HTML emails from React component trees; useful for dynamic newsletter generation in Node.js pipelines. URL: https://github.com/wix-incubator/mjml-react

- **Easy Email Editor** — Open-source (MIT) full-featured drag-and-drop email editor built on React and MJML; embeddable as a React component; supports MJML export, HTML export, and JSON schema for template storage. URL: https://open-source.easyemail.pro/

- **GrapesJS** — Open-source (BSD-3-Clause) page and email builder framework; includes MJML plugin for building responsive newsletters via drag-and-drop; widely used as a white-label email builder foundation. URL: https://grapesjs.com/

- **OpenAPI 3.1** — Used by newsletter platform APIs (Mailchimp, Beehiiv, Poppulo) to describe their REST management APIs; enables SDK code generation and automation integrations. URL: https://spec.openapis.org/oas/latest.html

- **JSON Schema** — Used for template configuration schemas in newsletter builders; defines the data structure for saved newsletter templates, content blocks, and personalisation merge tag definitions. URL: https://json-schema.org/

### Security & Authentication Standards

- **GDPR Article 6 — Lawful Basis for Processing** — Internal employee newsletters must have a lawful basis for processing employee email addresses; typically "legitimate interests" for mandatory company communications, but "consent" required for opt-in newsletters or tracking analytics. URL: https://gdpr-info.eu/art-6-gdpr/

- **GDPR Article 21 — Right to Object** — Employees in the EU may object to processing of their personal data for internal newsletter communications; platforms must support opt-out and honour objection requests promptly. URL: https://gdpr-info.eu/art-21-gdpr/

- **CAN-SPAM Act (US)** — US federal law governing commercial email; requires honest From and subject lines, physical mailing address, and working unsubscribe mechanism (honoured within 10 business days); FTC maximum civil penalty increased to $53,088 per violation effective January 17, 2025. URL: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business

- **CASL — Canada's Anti-Spam Legislation** — Stricter than CAN-SPAM; requires express or implied consent before sending commercial electronic messages; relevant for Canadian employee communications with promotional content. URL: https://crtc.gc.ca/eng/internet/anti.htm

- **SOC 2 Type II** — Required enterprise compliance certification for SaaS internal newsletter platforms; HubEngage, Staffbase, ContactMonkey, and Poppulo maintain SOC 2 Type II reports. URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2

- **SCIM 2.0 (RFC 7643/7644)** — System for Cross-domain Identity Management; used for automated employee roster synchronisation from HRIS (Workday, BambooHR, SAP SuccessFactors) to internal newsletter platforms for maintaining accurate employee distribution lists. URL: https://datatracker.ietf.org/doc/html/rfc7643

- **SAML 2.0 / OIDC** — Required for enterprise SSO integration ensuring internal newsletter platform access is controlled by the enterprise identity provider. URL: https://docs.oasis-open.org/security/saml/v2.0/

### MCP Server Specifications

AI-powered newsletter creation is integrating with the MCP ecosystem:

- **MJML MCP Server (Claude/Cursor/VS Code, March 2026)** — Brand-aware MJML-powered MCP server for generating responsive email templates from AI prompts; no API keys required; enables AI agents to create compliant HTML emails using MJML markup. URL: (community project, March 2026)

- **Mailchimp MCP Pattern** — Mailchimp's extensive API (Campaigns, Audiences, Templates, Automations) makes it a natural MCP integration target; community MCP servers exist for programmatic newsletter campaign management via AI agents.

- **AI Newsletter Generation Pattern (2025-2026)** — Emerging architecture: AI agents query internal knowledge bases (Confluence, Notion, SharePoint) via MCP, extract relevant content, and compose newsletter drafts in MJML via a template MCP server; editorial review then publishes via SendGrid/Amazon SES.

---

## Similar Products — Developer Documentation & APIs

### Mailchimp (Intuit)

- **Description:** Leading email marketing platform with a comprehensive REST API; strong for both external marketing and internal newsletters; 300+ integrations; Campaign, Audience, Template, Automation, and Analytics APIs; Transactional Email via Mandrill.
- **API Documentation:** https://mailchimp.com/developer/marketing/api/
- **SDKs/Libraries:** mailchimp-marketing (Node.js, Python, PHP, Ruby); @mailchimp/mailchimp_marketing (npm); Postman collection
- **Developer Guide:** https://mailchimp.com/developer/
- **Standards:** REST/JSON, OpenAPI 3.0, OAuth 2.0, API keys, Webhooks
- **Authentication:** API key (Basic Auth with user:apikey); OAuth 2.0 for partner integrations

### Beehiiv

- **Description:** Modern newsletter platform designed for creator-led newsletters and internal communications; API v2 (2025) supports subscriber management, publications, and automations; MJML-compatible email templates; Stripe integration for paid newsletters.
- **API Documentation:** https://developers.beehiiv.com/
- **SDKs/Libraries:** REST API (JSON); beehiiv-node (community); Zapier integration
- **Developer Guide:** https://developers.beehiiv.com/
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, Webhooks
- **Authentication:** API key; OAuth 2.0 for integrations

### Poppulo

- **Description:** Enterprise internal communications platform covering email, mobile, intranet, and digital signage; enterprise-grade APIs for campaign management, audience segmentation, and analytics; Developer Portal with full API documentation.
- **API Documentation:** https://developer.poppulo.com/
- **SDKs/Libraries:** REST API (JSON); webhook events; Salesforce/ServiceNow connectors
- **Developer Guide:** https://developer.poppulo.com/
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, Webhooks, SAML 2.0 (SSO)
- **Authentication:** OAuth 2.0; API token for programmatic access

### ContactMonkey

- **Description:** Internal email communication tool integrated with Outlook and Gmail; email tracking, sentiment analysis, and employee feedback collection (surveys, pulse checks) built into email sends; strong analytics for internal newsletters.
- **API Documentation:** https://www.contactmonkey.com/developers (requires account)
- **SDKs/Libraries:** REST API; Outlook add-in; Gmail extension
- **Developer Guide:** ContactMonkey developer portal (account required)
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, SAML 2.0 (Enterprise SSO)
- **Authentication:** OAuth 2.0 (Microsoft 365 / Google Workspace); API token

### Staffbase

- **Description:** Employee experience platform combining intranet, employee mobile app, email newsletters, digital signage, and SMS under one vendor; enterprise REST API for content management, user management, and analytics; SCIM 2.0 for HRIS sync.
- **API Documentation:** https://developer.staffbase.com/ (requires account)
- **SDKs/Libraries:** REST API; SCIM 2.0; Webhooks; HR system connectors (Workday, SAP SuccessFactors)
- **Developer Guide:** Staffbase developer portal (account required)
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, SAML 2.0, SCIM 2.0
- **Authentication:** OAuth 2.0; SAML 2.0 for SSO; API token for integrations

### MJML (Open Source Email Framework)

- **Description:** Open-source (MIT) responsive email markup language that compiles to email-client-compatible HTML; handles Outlook rendering quirks automatically; standard framework for programmatic newsletter template generation; CLI, Node.js API, and web playground.
- **API Documentation:** https://documentation.mjml.io/
- **npm Package:** mjml (https://www.npmjs.com/package/mjml)
- **SDKs/Libraries:** mjml (Node.js/npm); mjml-react (Wix Incubator); @faire/mjml-react; MJML online editor
- **Developer Guide:** https://documentation.mjml.io/
- **Standards:** MJML markup language, HTML5, CSS3, MIT licence; handles MIME multipart generation
- **Authentication:** Not applicable (local compilation library)

### React Email (Open Source)

- **Description:** Open-source (MIT) React-based email template library; JSX/TSX components for building HTML emails; supports previewing, exporting HTML, and testing across email clients; growing ecosystem of integrations with Resend, Nodemailer, Postmark, and other ESPs.
- **API Documentation:** https://react.email/docs
- **npm Package:** @react-email/components (https://www.npmjs.com/package/@react-email/components)
- **SDKs/Libraries:** @react-email/components; @react-email/render; @react-email/tailwind; React-based component ecosystem
- **Developer Guide:** https://react.email/docs
- **Standards:** React JSX/TSX, HTML5, CSS inline styles, MIT licence; renders to email-compatible HTML
- **Authentication:** Not applicable (local rendering library)

### Amazon SES (Email Sending Infrastructure)

- **Description:** AWS managed email sending service; highly cost-effective for high-volume internal newsletter delivery; REST API for sending, bounces, complaint handling, and suppression list management; integrates with MJML-generated HTML.
- **API Documentation:** https://docs.aws.amazon.com/ses/latest/APIReference/
- **SDKs/Libraries:** AWS SDK v3 (@aws-sdk/client-sesv2); AWS CLI; CloudFormation templates
- **Developer Guide:** https://docs.aws.amazon.com/ses/latest/dg/
- **Standards:** REST/JSON, OpenAPI (partial), AWS Signature V4 auth, SMTP, SPF/DKIM/DMARC automation
- **Authentication:** AWS IAM credentials (Access Key + Secret Key); IAM roles for EC2/Lambda

---

## Notes

- **MJML as the internal newsletter development standard**: MJML (MIT licence) is the universal developer standard for building responsive HTML email templates; it compiles to email-client-compatible HTML including Outlook-specific VML code; all internal newsletter builders should offer MJML import/export or generate MJML-compatible HTML.

- **MJML MCP Server (March 2026)**: An MJML-powered MCP server for Claude, Cursor, and VS Code enables AI agents to generate brand-aware responsive email templates from natural language prompts — a significant capability for AI-native internal newsletter generation workflows.

- **SPF/DKIM/DMARC enforcement (2024-2025)**: Gmail (February 2024) and Outlook (May 2025) now enforce SPF, DKIM, and DMARC for bulk senders; internal newsletter platforms using custom sending domains must configure these DNS records or face deliverability failures.

- **GDPR legitimate interests vs. consent**: Internal company newsletters sent to all employees typically rely on "legitimate interests" as the GDPR lawful basis; optional internal newsletters (e.g., hobby groups, optional updates) require explicit consent; analytics tracking (open rates, click tracking) requires separate consideration.

- **SCIM 2.0 for employee roster sync**: Enterprise internal newsletter platforms must support SCIM 2.0 to automatically sync employee distribution lists from HRIS systems (Workday, SAP SuccessFactors, BambooHR) — ensuring new hires are added and departed employees are removed without manual administration.

- **Open-source landscape**: MJML (MIT), React Email (MIT), GrapesJS (BSD-3-Clause), and Easy Email Editor (MIT) provide the complete open-source stack for building a custom internal newsletter editor; Amazon SES (usage-based) provides the most cost-effective email delivery infrastructure.
