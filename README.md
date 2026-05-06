# Internal Newsletter Builder

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open-source, AI-native internal newsletter platform with built-in readership analytics and AI writing assistance.

Internal Newsletter Builder is a self-hostable platform for organisations that need to run structured, measured employee communications programmes. It targets internal communications managers, HR business partners, and IT teams who today choose between expensive enterprise suites and minimal open-source mailing list managers, with little in between.

---

## Why Internal Newsletter Builder?

- Specialist commercial tools cluster between $8 and $30 per user per month, and enterprise intranet-plus-newsletter suites such as Axios HQ have historically run at $10,000+/month — putting modern internal comms tooling out of reach for many teams.
- Existing open-source options (listmonk, Keila, SendPortal) are email-only, lack visual design tooling, and offer no AI writing or analytics features comparable to commercial incumbents.
- Commercial AI-native platforms (Simpplr, Axios HQ) bundle newsletter capabilities into broader intranet suites that require heavy implementation and 100-user minimums.
- Over 40% of internal communications professionals cite analytics as the most valuable capability, yet self-hosted tools provide only basic open and click metrics.
- Enterprise newsletter tools depend on cloud SaaS data flows that conflict with privacy-first or regulated-industry deployment requirements.

---

## Key Features

### Authoring and Templates

- Visual or code-based email template builder for consistent branding
- Mobile-responsive HTML rendering aligned with standards used across major email clients
- Personalisation tokens for dynamic content insertion based on employee attributes
- Scheduled sends for specific dates and times

### Distribution and Audience

- Subscriber import, segmentation, and mailing list management
- HRIS integration to sync employee data for distribution lists
- Multi-recipient sends to departments, locations, or custom segments
- Unsubscribe and email preference management

### Delivery Infrastructure

- SMTP delivery with bounce handling
- Multichannel delivery roadmap covering email, SMS, and push notifications
- Role-based access controls and authentication

### Analytics

- Open rates, click-through rates, and engagement metrics
- Engagement heatmaps showing click patterns and attention areas
- Send-time and audience performance reporting

### AI Capabilities

- AI-drafted newsletter content from disparate inputs such as Slack threads, meeting notes, and OKR updates
- ML-suggested subject lines optimised for open rates
- Predictive send-time optimisation per employee
- ML-powered audience segmentation based on behaviour and role

---

## AI-Native Advantage

Where incumbents bolt AI onto existing editors, this project treats AI as a primary authoring surface: drafting newsletters from raw organisational signal (Slack, meeting notes, project updates, OKR dashboards) and rewriting for clarity. Personalisation operates at the content-block level so a single send produces different variants by role, location, and reading behaviour. Predictive send-time optimisation and sentiment analysis on aggregate readership feed back into the editorial calendar, replacing manual A/B testing cycles.

---

## Tech Stack & Deployment

The project targets self-hosted deployment as the primary mode, in line with comparable open-source projects such as listmonk, Keila, and SendPortal. Email rendering targets responsive HTML email standards (MJML, Email on Acid coverage). Delivery integrates with standard SMTP and API providers (SendGrid, Mailgun). Enterprise identity and directory sync use OAuth 2.0 and SCIM to pull employee data from Azure AD or Okta. WCAG 2.2 accessibility compliance is in scope for regulated-industry deployments.

---

## Market Context

The internal communications software market is valued at approximately $8.8 billion in 2026 and projected to reach $17.9 billion by 2035 at roughly 8% CAGR. Specialist newsletter tools price from ~$799/month for SMB platforms up to custom six-figure annual contracts at the enterprise tier. Primary buyers are internal communications managers and directors, HR business partners, executive assistants managing CEO communications, and IT teams evaluating Microsoft 365 or Google Workspace integrations.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
