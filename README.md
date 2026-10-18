# $TENCYL V2 (Cloudflare Worker + D1) - Deployed ✔️

One Worker serves the public site (server-rendered from D1), `/dashboard`, and the API. **Nothing here touches your production database until you run the commands below yourself.**

## Upgrading an existing deployment (safe order)
1. Back up first: Dashboard → JSON Data → *Backup incl. inquiries*.
2. `wrangler d1 execute stencyl --remote --file=migrations/0002_v2.sql` — run **once**. Additive only: new columns on `inquiries` (existing rows get status `New`, notification status `unknown`) and new tables `versions`, `events`. Not repeatable (SQLite has no `ADD COLUMN IF NOT EXISTS`); a "duplicate column" error means it was already applied. Rollback notes are in the file.
3. `wrangler deploy`.
The Worker also runs *before* the migration (inquiries keep working; new features show "migration not applied"), so step order is forgiving. Existing live/draft/client content is read as-is and gets safe defaults for the new fields.
Fresh install: `schema.sql` then `migrations/0002_v2.sql`.

## Environment (Worker secrets — `wrangler secret put NAME`)
`ADMIN_PASSWORD` (12+ chars) · `RESEND_API_KEY` · `DISCORD_WEBHOOK_URL`. Never sent to the browser or included in exports. Local dev: `.dev.vars`.

## What's new
**Public:** mobile nav + active-section indicator + scroll progress, dark/light toggle (dark default; saved choice wins), loading state, availability chip (dashboard-controlled, off by default), project search/category filters/sorting/featured badges/detail dialog/copy-link (`/#project-<slug>`), Services, Process timeline, FAQ accordion, estimator (suggests one of your visible tiers, clamped to ₹250–₹1,000, can pre-fill the inquiry form), copy-brief, submission reference number, optional public contact email + copy button, back-to-top, offline notice, dynamic © year, custom 404, `robots.txt`, `sitemap.xml`, canonical + Open Graph tags, favicon/apple-touch/OG image generated from your logo. Fixed: nav links `#hero`/`#projects` now work.
**Dashboard:** real overview metrics + activity; Services/Process and FAQ managers (with "Add starter content"); project search/slug/details/bulk visibility; Inquiry CRM (status, tags, internal notes, filters, link to client draft, copy, CSV/JSON export, delete with confirm, per-channel email/Discord status, safe retry); client search/filter, 5 statuses, archive/restore, linked inquiries; Version History (snapshot per publish/restore, 50 kept, diff vs live or between versions, preview, restore as a *new* version); Analytics; Site Health; Ctrl/Cmd+S; save conflicts are rejected (revision check) instead of overwriting.

## Notes on behaviour
- Notification retry atomically claims a failed channel (`failed|pending → sending`) and sends Resend with an idempotency key, so it cannot duplicate the inquiry or an already-sent channel. If a Worker crashes mid-send a channel can stay `sending`; reset with `UPDATE inquiries SET email_status='failed' WHERE id='…'`.
- Pre-V2 inquiries have notification status `unknown` and are not retryable (we can't know what was delivered).
- Inquiry rate limit is now 5/hour/IP and only counts well-formed submissions.
- Analytics: counters per day/event/page/referrer-type/device only. No IPs, cookies or URLs stored; Do Not Track respected; only the live site reports (previews don't).
- Internal notes are stored only on the inquiry row; never in configs/exports. They appear in the inquiry export only if you tick "include internal notes". Content exports never contain inquiries; only *Backup* does.
- SEO: update your SEO description in the dashboard (stored copy has a typo and the older text). Meta tag text is set from there.

## JSON schema (v1, additive)
Config adds optional `site.availability{enabled,text}`, `site.contactEmail`, `projects[].slug/details`, `services[]`, `process[]`, `faq[]` (all `{…,visible,order}`). Older files without them still import. Pricing: whole numbers 250–1000, enforced on every write and import.

## Needs extra setup / not included
- **Media library / R2 uploads: not built.** Thumbnails remain URL references. Add an R2 binding and an upload route before offering uploads.
- Not built: dedicated project detail routes (dialog + hash links instead), drag-and-drop ordering (↑/↓ buttons), autosave, undo/redo, theme-color/3D-intensity/section-order editor, project-only JSON import/export, light theme for the dashboard (public site has it).
- Email requires a verified `stencylstudio.com` domain in Resend.

## Tests (`npm test`, Node 22)
Unit tests for the validator, pricing limits, CSV escaping, diff, normalization; integration tests run the real Worker against an in-memory SQLite stand-in for D1: auth, CSRF, login rate limit, optimistic concurrency, draft→publish→restore, client isolation and share-link hashing/revocation, inquiry flow, notification failure+retry, CRM filters, exports without secrets, import validation, analytics privacy, public routes.
**Not tested:** real Cloudflare D1/Workers runtime, real Resend/Discord delivery, browser rendering of the public site or dashboard (including the 3D scene and new UI), mobile layouts, R2.
