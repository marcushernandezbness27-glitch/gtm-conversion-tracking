# Case Study: GTM + GA4 Conversion Tracking — Stellapura Agent Portal

**Role:** Sole implementer (GTM setup, GA4 configuration, conversion event design, Worker code integration)  
**Stack:** Google Tag Manager, Google Analytics 4, Hono JSX (Cloudflare Workers), TypeScript

---

## The problem

The Stellapura payroll platform had no visibility into how agents were finding and completing the registration funnel. The site was live with real users, but there was no data on page views, drop-off points, or conversion rates — no way to know if the registration flow was working or where it was breaking.

---

## What was built

A full GTM + GA4 tracking setup on a live Cloudflare Workers site:

1. GTM container (`GTM-56QMHVF2`) installed across all pages via shared layout components
2. GA4 property connected via GTM — no direct script install, all managed through the container
3. Custom conversion event (`agent_registration`) firing on registration form submission
4. Verified live in GA4 Realtime reports

---

## Implementation decisions worth noting

### GTM installed via server-rendered JSX layout
The Stellapura site runs on Cloudflare Workers using Hono's JSX renderer. Unlike a static HTML site, inline scripts in JSX are HTML-escaped by default — which silently breaks the GTM snippet. The fix: `dangerouslySetInnerHTML` on the script element, which bypasses JSX escaping and outputs the raw JavaScript GTM requires. Applied to both the admin layout and public layout so all pages are covered.

### GA4 managed through GTM — not direct install
GA4 was added as a tag inside GTM rather than pasting the gtag snippet directly into the codebase. This keeps all tag management in one place — adding or modifying analytics tags in future requires no code changes or redeployment.

### Conversion event scoped to the registration route
The `agent_registration` trigger fires on form submissions on `/register` only, not all forms sitewide. This prevents false positives from other form interactions (login, search) polluting conversion data.

---

## Result

Live conversion tracking on a production Cloudflare Workers site with verified GA4 data flow — page views and agent registration completions tracked in real time. Adding new conversion events (portal login, payroll request submission) requires no code changes — new tags are added in GTM and published.

---

## Honest gaps

- **No Google Ads conversion tag:** the setup tracks events in GA4 but does not yet import conversions into Google Ads for ROAS reporting. A production paid acquisition setup would add a Google Ads Conversion Tracking tag alongside the GA4 event.
- **No consent mode:** the current setup fires tags without a consent banner. A production EU-facing deployment would require CMP integration and GTM consent mode configuration.
- **No Meta Pixel:** not installed yet — would follow the same GTM pattern (Meta Pixel tag + custom event trigger).

---

*Implemented on a live production site (stellapura.maillab.live) as portfolio evidence for conversion tracking and tag management work.*
