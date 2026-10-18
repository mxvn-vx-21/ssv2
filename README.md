# \$TENCYL --- Website & Admin Dashboard

**Small Studio. Stunning Sites.**

The official website and private content-management dashboard for
**\$TENCYL**, an independent website development studio focused on
minimalist websites, GlassUI, and interactive web experiences.

-   **Website:** https://stencylstudio.com
-   **Dashboard:** https://stencylstudio.com/dashboard
-   **Hosting:** Cloudflare Workers
-   **Database:** Cloudflare D1
-   **Email notifications:** Resend
-   **Discord notifications:** Discord webhook

> This repository contains the application source. Keep production
> credentials in Cloudflare secrets; never commit secrets or include
> them in exported JSON files.

## Features

### Public website

-   Responsive, monochrome design with glass-inspired UI.
-   Dark/light theme toggle with a saved preference.
-   Mobile navigation, active-section indicator, and scroll progress.
-   Interactive project gallery with search, category filters, sorting,
    featured badges, project details, and copy-link support.
-   Services, process timeline, and FAQ sections.
-   Project price estimator that uses visible pricing tiers and keeps
    estimates within **₹250--₹1,000**.
-   Inquiry form with validation, submission reference, and copyable
    project brief.
-   Resend email notifications and Discord webhook notifications.
-   SEO metadata, canonical and Open Graph tags, favicon assets,
    `robots.txt`, and `sitemap.xml`.
-   Custom 404 page, back-to-top control, and responsive layouts.

### Private dashboard

-   Password-based admin login with session handling, CSRF protection,
    and login rate limiting.
-   Edit website content and manage projects, pricing, services, process
    steps, and FAQs.
-   Save drafts and publish changes explicitly.
-   Independent client drafts with isolated configurations and private,
    expiring, revocable review links.
-   Desktop, tablet, and mobile preview modes.
-   JSON import/export for supported website content and backups.
-   Inquiry CRM with statuses, tags, internal notes, filters,
    client-draft links, and CSV/JSON export.
-   Separate email and Discord delivery status with retry support for
    eligible failed notifications.
-   Version history with snapshots, comparisons, previews, and
    restore-as-a-new-version.
-   Analytics designed to avoid storing raw IP addresses.
-   Site Health page and recent activity.
-   Revision checks that reject conflicting saves instead of silently
    overwriting newer edits.

## Architecture

A single Cloudflare Worker serves the public website, `/dashboard`, and
the application API. Public content is rendered from the configuration
stored in D1.

``` text
Browser
  └── Cloudflare Worker
       ├── Public website
       ├── Private dashboard
       ├── API and validation
       ├── Authentication and sessions
       ├── Resend email notifications
       ├── Discord webhook notifications
       └── Cloudflare D1
            ├── Site configuration
            ├── Projects and pricing
            ├── Client drafts and review links
            ├── Inquiries
            ├── Sessions and rate limits
            ├── Version history
            └── Analytics events
```

### Repository structure

``` text
.
├── src/
│   ├── worker.js          # Worker routes, API, auth, persistence
│   └── lib.js             # Validation and shared helpers
├── public/
│   ├── index.tpl          # Public site template
│   ├── dashboard.tpl      # Admin dashboard interface
│   ├── favicon.png
│   ├── apple-touch-icon.png
│   └── og.png             # Social preview image
├── migrations/
│   └── 0002_v2.sql        # Additive V2 database migration
├── tests/
│   ├── lib.test.mjs
│   └── worker.test.mjs
├── schema.sql             # Initial database schema
├── wrangler.toml          # Cloudflare Worker and D1 configuration
├── package.json
└── README.md
```

## Pricing constraint

The website's pricing is intentionally limited to **₹250--₹1,000**.

Pricing values must be whole numbers within that range. Validation is
applied when content is saved or imported; do not bypass the validation
in new features or API endpoints.

## Configuration and secrets

The Worker expects these Cloudflare secrets:

  Secret                  Purpose
  ----------------------- -----------------------------------------------
  `ADMIN_PASSWORD`        Password used to access the private dashboard
  `RESEND_API_KEY`        Sends inquiry notification emails
  `DISCORD_WEBHOOK_URL`   Sends inquiry notifications to Discord

Set secrets from a terminal with Wrangler:

``` bash
npx wrangler secret put ADMIN_PASSWORD
npx wrangler secret put RESEND_API_KEY
npx wrangler secret put DISCORD_WEBHOOK_URL
```

Enter each value when prompted. Do not place secret values in source
files, frontend JavaScript, Git commits, screenshots, or JSON content
exports.

Use a strong, unique admin password. The sender address
`no-reply@stencylstudio.com` requires the domain to be verified and
configured in Resend before email delivery can work.

For local development, use a local `.dev.vars` file if needed and keep
it out of Git.

## Cloudflare setup

The production deployment is already live. The steps below are for a new
environment or a deliberate redeployment---not an instruction to
recreate or reset the existing production database.

### Requirements

-   A Cloudflare account.
-   A Cloudflare D1 database.
-   Wrangler CLI and a supported Node.js version for local
    development/deployment.
-   Resend account and verified sending domain.
-   Discord webhook URL for the target channel.

### 1. Install dependencies

``` bash
npm install
```

The project uses Node's built-in test runner. There are no package
dependencies declared in the current `package.json`.

### 2. Configure the D1 database

Update `wrangler.toml` with the actual D1 database ID for your
environment. Do not leave the placeholder `REPLACE_WITH_D1_ID`.

The configuration expects a D1 binding named `DB` and a database named
`stencyl`.

For a **fresh database only**, apply the initial schema and then the V2
migration:

``` bash
npx wrangler d1 execute stencyl --remote --file=schema.sql
npx wrangler d1 execute stencyl --remote --file=migrations/0002_v2.sql
```

### 3. Existing V1 installation

If upgrading an existing V1 database, **back it up first**. The V2
migration is additive, but it is not idempotent: running it a second
time can fail with a duplicate-column error.

For the original V1-to-V2 upgrade, the migration command is:

``` bash
npx wrangler d1 execute stencyl --remote --file=migrations/0002_v2.sql
```

Run it only if it has not already been applied. Do not run `schema.sql`
against an existing production database as a substitute for a migration.

### 4. Set secrets and deploy

After verifying the Worker configuration and database binding:

``` bash
npx wrangler secret put ADMIN_PASSWORD
npx wrangler secret put RESEND_API_KEY
npx wrangler secret put DISCORD_WEBHOOK_URL
npx wrangler deploy
```

Attach `stencylstudio.com` through the Cloudflare dashboard if it is not
already configured. Verify the domain in Resend and configure the
required DNS records before relying on outbound email.

**Never run database commands against production unless you have
confirmed the target database and have a current backup.**

## Testing

Run the automated test suite:

``` bash
npm test
```

The V2 development run reported **21 passing automated tests**. These
include validation, pricing constraints, CSV escaping, configuration
normalization, authentication/CSRF behavior, save-conflict handling,
publishing/restoring, client-draft isolation, inquiry handling,
notification failure/retry, exports, import validation, analytics
privacy, and public routes.

The integration tests use an in-memory SQLite stand-in for D1 while
exercising Worker code. They do **not** replace testing against the
actual Cloudflare environment.

After deploying, manually verify:

-   Public pages and mobile navigation.
-   Dashboard login and logout.
-   Saving, previewing, and publishing a draft.
-   Client preview isolation and review-link revocation.
-   JSON export and import.
-   A real inquiry submission.
-   Resend email delivery and Discord notification delivery.
-   Notification retry behavior.
-   Site Health and analytics.
-   Browser console errors and mobile performance.

Do not assume that a successful local test proves that production email,
Discord, D1, or browser rendering is working.

## Known limitations

The current implementation has these known limitations:

-   Project thumbnails use URLs; an R2-backed media upload library is
    not included.
-   Projects open in a detail dialog with a shareable hash link rather
    than dedicated project routes.
-   Drag-and-drop ordering is not implemented; ordering controls use
    simpler interactions.
-   Autosave and undo/redo are not implemented.
-   Advanced visual controls for theme colors, 3D intensity, and section
    order are not included.
-   Project-only JSON import/export is not implemented.
-   The public site supports dark/light mode; the dashboard does not yet
    have a separate light theme.
-   Production behavior depends on correct Cloudflare bindings, database
    migrations, Resend verification, and Discord webhook configuration.

Keep this list updated as features are implemented.

## Security notes

-   Protect `/dashboard` and all admin APIs with server-side
    authentication and authorization.
-   Preserve CSRF protection, rate limiting, validation, and secure
    session handling.
-   Keep private client drafts and inquiry notes out of public
    responses.
-   Do not export environment secrets, sessions, passwords, or webhook
    URLs.
-   Validate all imported JSON on the server.
-   Keep dependencies and Cloudflare configuration maintained.
-   Back up content before schema migrations or major changes.

## Content and data management

Use the dashboard to edit content, save drafts, preview changes, and
publish intentionally. Use JSON export to keep a portable content
backup. Exporting JSON does **not** itself deploy the website;
deployment remains a separate operation.

Client drafts should remain isolated from the live public configuration.
Review links should be shared only with their intended recipients and
revoked when no longer needed.

## License

No open-source license is specified in this repository. Unless a license
is added, assume the source is provided for private project use and that
standard copyright restrictions apply.
