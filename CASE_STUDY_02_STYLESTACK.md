# Case Study: Multi-Service Conversion Tracking — StyleStack Platform

**Role:** Sole implementer (platform architecture, GTM setup, Meta Pixel + Conversions API, GA4, double opt-in email flow, Worker code)
**Stack:** Hono + TypeScript (Cloudflare Workers), Cloudflare KV, Google Tag Manager, Meta Pixel + Conversions API, Google Analytics 4, Resend

---

## The problem

Case Study 1 (Stellapura) tracked a single conversion event on a single-purpose site — a reasonable starting point, but it left open questions a real platform raises: what happens when there's more than one product to track? What happens when a "signup" isn't really confirmed until the person proves they own the email address? And what happens when the browser Pixel alone misses conversions blocked by ad blockers or Safari's tracking protection?

StyleStack is a multi-service membership platform (one brand, multiple independently-launchable products, modeled on Autodesk/JetBrains-style suites) — a deliberately harder tracking problem than a single form on a single page.

---

## What was built

1. A GTM container (`GTM-W76Z9K4K`) dedicated to this project, installed via a shared Hono JSX layout using the same `dangerouslySetInnerHTML` escaping fix from Case Study 1
2. A **single Meta Pixel** (`StyleStack Pixel`) shared across every service on the platform, instead of one pixel per product — services are distinguished by an event parameter (`content_name`), not by separate tags or separate pixels
3. A real **double opt-in signup flow**: submitting the form records a `pending` signup and sends a confirmation email (via Resend); the signup only becomes `confirmed` when that link is clicked. Two distinct conversion events map to these two states — `Lead` (unverified intent) and `CompleteRegistration` (verified) — so ad reporting doesn't conflate typo'd/fake emails with real signups
4. **Server-side Meta Conversions API**, sent in parallel with the browser Pixel for the same two events, with a shared `event_id` so Meta deduplicates the browser and server events into one conversion instead of double-counting
5. A custom domain (`www.stylists.space`) rather than a `*.workers.dev` subdomain, with DNS migrated to Cloudflare specifically to get reliable programmatic DNS management for the email-sending subdomain
6. **Duplicate-submission handling** that distinguishes three states instead of treating every submission as identical: a brand-new signup, a resubmission while still unconfirmed (resends the confirmation email, tells the visitor so), and a resubmission of an already-confirmed email (no new email, no re-fired conversion event, different on-page messaging) — so a repeat submission from someone who already converted doesn't inflate Lead counts in Ads Manager
7. Google Analytics 4 wired in alongside Meta, reusing the exact same dataLayer contract and GTM triggers (`generate_lead` / `sign_up` events) — no new code, just two additional GTM tags

---

## Implementation decisions worth noting

### One Meta Pixel for N services, not N pixels
A separate Pixel per product doesn't scale — every new service would mean a new pixel, new tags, and no unified view of platform-wide conversion performance. Instead, every service's dataLayer event carries a `service` key, mapped in GTM to a Data Layer Variable and passed as the Pixel's `content_name` parameter. Adding a second service (ScheduleFlow, currently a stub) to conversion tracking requires zero new GTM tags — just a new event name matching the existing `.*_signup` / `.*_confirmed` regex triggers.

### Double opt-in, and why it's two Meta events, not one
The original single-step "waitlist" design (Case Study 1's approach) let anyone submit any email — including ones they don't own — and counted it as a conversion immediately. StyleStack's signup writes a `pending` KV record and only flips to `confirmed` after the recipient clicks a single-use, time-limited link emailed via Resend. Firing `Lead` on submission and `CompleteRegistration` only on confirmation means Meta's ad optimization (and any future reporting) can be pointed at the verified signal instead of raw, unverified form-fills.

### Server-side Conversions API, deduplicated against the browser Pixel
Browser-only tracking misses conversions blocked by ad blockers or privacy features like Safari's Intelligent Tracking Prevention — a real, common gap for real ad accounts. Adding server-side Conversions API events (from the same Worker that already processes the signup and confirmation) closes that gap, but naively running both channels risks double-counting. The fix: the server generates a UUID `event_id` per conversion and returns it to the browser (via the signup API response, or embedded directly in the server-rendered confirmation page); the browser includes that same ID in its dataLayer push, which GTM passes to the Pixel's `eventID` field. Meta merges any browser + server event pair sharing an `event_name` and `event_id` within 48 hours into a single conversion.

### Skipped Meta's Conversions API Gateway product
Meta's UI defaults to offering a "Conversions API Gateway" setup — routing pixel data through a third-party-hosted gateway (in this case, a provider called Datahash) requiring a separate account and API key. That's a bigger, different integration than a direct server-to-server call and introduces a vendor dependency with no clear benefit here. Used Meta's plain manual access-token setup instead, calling `graph.facebook.com` directly from the Worker.

---

## Result

Two real, working Meta conversion signals (`Lead` and `CompleteRegistration`), deduplicated between browser Pixel and server-side Conversions API, plus the equivalent GA4 events (`generate_lead`, `sign_up`) — all driven by one dataLayer contract, verified end-to-end against a live custom domain rather than a demo environment. Resubmitting an already-confirmed signup no longer inflates the Lead count in either destination. A second service can be added to tracking with zero new GTM configuration beyond a new event name matching the existing trigger regex. The full architecture (dataLayer contract, GTM tag structure, CAPI dedup mechanism) is documented in the repo's `GTM_SETUP.md` for anyone extending it.

---

## Honest gaps

- **No consent mode:** tags fire without a consent banner, same gap as Case Study 1. A production EU-facing deployment needs CMP integration and GTM consent mode.
- **No Google Ads conversion import:** GA4 now tracks the same two events, but nothing feeds them into Google Ads for ROAS reporting — the same gap Case Study 1 left open, still unaddressed here.
- **CAPI dedup requires manual GTM tag configuration** (`eventID` parameter, `DLV - event_id` variable) — this isn't enforced by code, so a future tag added without following the documented pattern could silently reintroduce double-counting. Worth a periodic Test Events check rather than a one-time verification.
- **No cron-based cleanup for stale KV state beyond TTL:** pending (unconfirmed) signups expire via Cloudflare KV's TTL, but there's no reporting on how many signups never confirm — a real production setup would want that as a funnel metric, not just silent expiry.
- **A narrow KV-consistency race can undo a confirmation:** Workers KV reads can be served from an up-to-60-second edge cache. Resubmitting the signup form within that window of confirming can read the pre-confirmation state and silently write the record back to "pending" — found and verified live while testing the duplicate-signup detection feature (see "What was built" above). Closing this fully needs a strongly-consistent store (a Durable Object) for this per-email state; accepted as a rare edge case rather than adding that infrastructure for a portfolio project.

---

*Implemented on a live, custom-domain-hosted Cloudflare Workers site (www.stylists.space) as portfolio evidence for conversion tracking and multi-service tagging architecture work.*
