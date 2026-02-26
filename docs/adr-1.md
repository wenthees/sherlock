# ADR-001: Technology Stack for Dong Vui Feedback System

| Field       | Detail                                      |
|-------------|---------------------------------------------|
| **Status**  | 🟡 Proposed                                 |
| **Date**    | 2026-02-21                                  |
| **Authors** | Alex                                        |
| **Deciders**| Chris, Alen                                 |

---

## Context

Dong Vui Food Court (Mui Ne, Vietnam) operates a multi-kitchen food court serving a high-turnover tourist population. The existing customer-facing website is hosted on **Wix**. We are building a feedback system that allows guests to rate individual dishes per kitchen, and allows kitchen managers to monitor quality trends.

Key constraints and observations:

- The existing site runs on **Wix** — any solution must account for integration limits (Wix does not allow arbitrary server-side code or custom backends natively).
- The guest-facing feedback flow is a **mobile-first web app** accessed via QR code at tables.
- The database schema already exists as a CSV export with fields: `Kitchen Name`, `Item Name`, `Description`, `Category`, `Price`, `Is Active`, and a `Prices (JSON)` field suggesting price variants (e.g. dual pricing for tourists vs. locals).
- Guests are **international tourists** — the UI must support multiple languages (EN, VI, ZH, KR, RU, FR confirmed in design).
- Traffic is **bursty and low-volume** — no need for enterprise-scale infrastructure.
- The team is likely small; operational simplicity is a priority.

---

## Decision Drivers

1. **Wix compatibility** — integration must be achievable without breaking Wix's hosting model.
2. **Mobile UX quality** — feedback must work fast on a tourist's phone, on 4G, in bright sunlight.
3. **Low operational burden** — no DevOps team; managed services preferred.
4. **Data ownership** — feedback data should be exportable and owned by the business.
5. **Cost** — solution should remain affordable at low traffic volumes.
6. **Iteration speed** — design is still evolving; framework should not lock in UI decisions early.

---

## Options Considered

### Option A — Wix Velo (native)

Build everything inside Wix using Wix Velo (formerly Corvid): frontend pages with Wix CMS as the database, and Velo backend functions for logic.

**Pros:**
- Zero infrastructure; stays entirely within Wix ecosystem.
- Wix CMS can store feedback records natively.
- No CORS issues; same domain as existing site.

**Cons:**
- Velo's data APIs are slow and rate-limited for dynamic reads.
- Very limited control over UI/UX — the feedback flow we've designed (card layouts, animated tag drawers, thumb interactions) would be extremely difficult to replicate in Wix's page editor.
- No native i18n support; multi-language would require manual duplication of pages.
- Tightly coupled to Wix; migrating later is painful.
- Poor developer experience for complex state management.

**Verdict:** ❌ Not recommended. The custom UX we need exceeds what Velo can reasonably deliver.

---

### Option B — Static SPA (plain HTML/CSS/JS) + External Backend

Build the guest feedback app as a standalone static site (pure HTML, CSS, vanilla JS or a lightweight bundler). Host separately (e.g. Cloudflare Pages, Netlify). Connect to a simple backend API.

**Pros:**
- Maximum control over UX — the designs we've built already use this approach.
- Deployable to Cloudflare Pages for free; fast global CDN.
- Works as an `<iframe>` or redirect from Wix.
- No build tooling required; easy to hand off.

**Cons:**
- State management in vanilla JS becomes unwieldy as complexity grows.
- No component reuse without a framework.
- Multi-language strings require a manual i18n solution.

**Verdict:** ✅ Viable for MVP. Good starting point given the designs are already in plain HTML.

---

### Option C — React SPA + Supabase + Wix Embed (Recommended)

Build the guest feedback app as a **React SPA** (via Vite). Host on **Cloudflare Pages** or **Netlify**. Use **Supabase** as the backend (Postgres database, REST/realtime API, auth). Embed in Wix via HTML iframe embed widget or link out via QR code.

**Pros:**
- React gives us componentised dish cards, language context, and state management — all already needed.
- Supabase provides a managed Postgres database with a REST API, real-time subscriptions (for a live kitchen dashboard), and row-level security with no backend code to write.
- The existing CSV schema maps cleanly to Supabase tables.
- Wix integration: Wix supports embedding external HTML/iframes via the **Embed HTML** element. The feedback app can live at `feedback.dongvui.com` and be embedded or linked directly from the Wix site.
- Supabase has a generous free tier; cost is near-zero at this volume.
- The kitchen manager dashboard (future) can be a separate protected React route querying the same Supabase instance.

**Cons:**
- React adds build complexity vs. plain HTML.
- Supabase is a third-party dependency; if it changes pricing, migration is required.
- Wix iframe embeds can have height/scroll quirks on mobile — QR code direct link may be cleaner UX than embedding.

**Verdict:** ✅✅ **Recommended.** Best balance of UX capability, operational simplicity, and Wix compatibility.

---

### Option D — Next.js + Vercel + Supabase

Same as Option C but using **Next.js** for server-side rendering and API routes, deployed on **Vercel**.

**Pros:**
- SSR improves initial load performance and SEO.
- API routes can proxy Supabase calls, hiding credentials from client.

**Cons:**
- SSR is largely unnecessary for a feedback app accessed by QR code — there is no SEO value.
- Next.js adds significant complexity (routing conventions, server components) for a project this size.
- Overkill for current scope.

**Verdict:** ❌ Deferred. Reconsider if the project grows to include a public-facing menu or SEO-indexed pages.

---

## Decision

**Adopt Option C: React (Vite) + Supabase + Cloudflare Pages, with Wix integration via direct link / iframe embed.**

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                   Wix Website                        │
│  (existing, dongvui.com)                            │
│                                                      │
│  ┌──────────────────────────────────┐               │
│  │  QR Code / Button → feedback URL │               │
│  └──────────────────────────────────┘               │
└─────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│         React SPA  (Vite + React 18)                │
│         feedback.dongvui.com                        │
│         Hosted: Cloudflare Pages                    │
│                                                      │
│  Pages:                                             │
│  /              → Kitchen selector                  │
│  /kitchen/:id   → Dish rating page                 │
│  /admin         → Kitchen manager dashboard (auth) │
│                                                      │
│  i18n: react-i18next (EN, VI, ZH, KR, RU, FR)     │
└─────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────┐
│              Supabase (managed Postgres)             │
│                                                      │
│  Tables:                                            │
│  kitchens       (id, name, cuisine, emoji, active)  │
│  items          (id, kitchen_id, name, desc,        │
│                  category, price, is_active, ...)   │
│  feedback       (id, item_id, rating, tags,         │
│                  comment, lang, created_at,         │
│                  session_id)                        │
│                                                      │
│  Auth: Supabase Auth (admin dashboard only)        │
│  Realtime: feedback stream → dashboard             │
└─────────────────────────────────────────────────────┘
```

---

## Database Mapping

The existing CSV export maps to Supabase tables as follows:

| CSV Field         | Supabase Column          | Notes                                      |
|-------------------|--------------------------|--------------------------------------------|
| `ID`              | `items.id`               | Use as primary key                         |
| `Kitchen Name`    | `kitchens.name`          | Normalise into separate `kitchens` table   |
| `Item Name`       | `items.name`             |                                            |
| `Description`     | `items.description`      |                                            |
| `Category`        | `items.category`         | Useful for grouping on dish page           |
| `Price`           | `items.price_vnd`        |                                            |
| `Low Price`       | `items.price_low_vnd`    | For dual pricing                           |
| `High Price`      | `items.price_high_vnd`   | For dual pricing                           |
| `Use Dual Pricing`| `items.dual_pricing`     | Boolean                                    |
| `Prices (JSON)`   | `items.prices_json`      | Store as `jsonb`; parse for complex tiers  |
| `Is Active`       | `items.is_active`        | Hide inactive items from feedback UI       |

---

## Wix Integration Strategy

Wix does not support custom backends or Node.js servers natively. Integration options, in order of preference:

1. **QR Code → Direct URL** *(recommended)* — Table QR codes point directly to `feedback.dongvui.com`. No Wix involvement at runtime. Clean, fast, no iframe quirks.
2. **Wix HTML Embed** — Paste an `<iframe src="https://feedback.dongvui.com" ...>` into a Wix page via the Embed HTML element. Works but mobile scroll behaviour inside iframes can be unreliable.
3. **Wix Button Link** — A button on the Wix site links out to the feedback URL in a new tab. Simplest, no embedding complexity.

Menu data currently lives in the CSV; it should be migrated to Supabase as the source of truth. The Wix site's menu pages (if any) can either read from Supabase via a public API endpoint or continue to be maintained manually in Wix — this is a separate decision (see ADR-002 candidate).

---

## Consequences

**Positive:**
- Full control over UX — the animated, multilingual feedback flow we've designed is implementable without compromise.
- Feedback data is in a real relational database; analytics and reporting are straightforward.
- Kitchen manager dashboard is a natural extension of the same stack.
- Cloudflare Pages has zero cold-start latency; fast load for tourists on mobile.

**Negative / Risks:**
- React adds a build step; contributors need Node.js familiarity.
- Supabase is a managed third-party; API key management and RLS policies must be set up correctly to prevent data leakage.
- The feedback app and Wix site are separate deployments; content (e.g. menu changes) must be updated in Supabase, not just Wix.

---

## Open Questions

- [ ] Should the admin dashboard be part of this app or a separate Supabase Studio view initially?
- [ ] How should `session_id` be generated — device fingerprint, or a short-lived token to prevent duplicate voting within a session?
- [ ] Is dual pricing (tourist / local) surfaced in the feedback UI, or only in the admin view?
- [ ] Who manages Supabase credentials and deployments? Is there a technical owner on the team?
- [ ] Should inactive items (`is_active = FALSE`) be hidden completely or shown as "unavailable" on the rating page?

---

## References

- [Wix Velo Documentation](https://support.wix.com/en/article/velo-getting-started)
- [Wix Embed HTML Element](https://support.wix.com/en/article/wix-editor-embedding-a-site-or-a-widget)
- [Supabase Quickstart](https://supabase.com/docs/guides/getting-started)
- [Cloudflare Pages](https://pages.cloudflare.com/)
- [react-i18next](https://react.i18next.com/)
- Existing UI designs: `dong-vui-landing.html`, `dong-vui-dishes.html`
