# StyleStack Demo — Setup & Deployment Guide

Extends the existing Stellapura GTM/GA4 portfolio (see `CASE_STUDY.md`) by adding:
- A second live site on Cloudflare Pages (static HTML)
- A new, dedicated GTM container for this site (separate from `GTM-56QMHVF2`)
- Meta Pixel installed via GTM — fills the "Honest gap" noted in the Stellapura case study
- Conversion event (`waitlist_signup`) triggering a Meta Lead event on form submission

---

## Step 1 — Deploy to Cloudflare Pages

```bash
cd d:\_freelancing\Marcus\Portfolio\gtm-conversion-tracking

npx wrangler pages deploy stylestack-demo --project-name=stylestack-demo
```

Wrangler will prompt you to log in on first run. After deployment you'll receive a live URL like:
`https://stylestack-demo.pages.dev`

Test it loads before continuing.

---

## Step 2 — Create a New GTM Container

> Do NOT reuse GTM-56QMHVF2 — that container belongs to the Stellapura site.
> Using two unrelated sites in one container pollutes tag lists and makes both harder to read.

1. Go to [tagmanager.google.com](https://tagmanager.google.com)
2. Open your existing account → **Create Container**
3. Container name: `stylestack-demo.pages.dev`
4. Target platform: **Web**
5. Click **Create** → accept terms
6. Note the new Container ID (format: `GTM-YYYYYYY`)

Then update the placeholder in `stylestack-demo/index.html`:
- Replace **both** occurrences of `GTM-XXXXXXX` with your new container ID (once in `<head>`, once in the `<noscript>` tag)
- Redeploy: `npx wrangler pages deploy stylestack-demo --project-name=stylestack-demo`

---

## Step 3 — Create a Meta Business Manager Account + Pixel

> Use a **separate** Meta Business Manager account from any real client accounts (Stellapura, etc.).
> Portfolio-purpose accounts should be isolated from client data.

1. Go to [business.facebook.com](https://business.facebook.com) → create a new personal Business Manager account
2. Under **Events Manager** → **Connect Data Sources** → **Web**
3. Choose **Meta Pixel** → name it `StyleStack Demo`
4. Skip the automatic setup wizard — you'll install via GTM instead
5. Note the **Pixel ID** (a numeric string, e.g. `1234567890123456`)

---

## Step 4 — Add Meta Pixel Tag in GTM

In your new GTM container (`GTM-YYYYYYY`):

### 4a — Meta Pixel Base Code Tag (All Pages)

1. **Tags → New**
2. Tag type: search for **Meta Pixel** (GTM's built-in template)
   - If not available, use **Custom HTML** and paste the Meta Pixel base code with your Pixel ID
3. Pixel ID: `[your Pixel ID from Step 3]`
4. Event to fire: `PageView`
5. Triggering: **All Pages**
6. Name: `Meta Pixel — Page View`
7. **Save**

### 4b — Meta Pixel Lead Event Tag (Waitlist Signup)

1. **Tags → New**
2. Tag type: **Meta Pixel** (or Custom HTML)
3. Pixel ID: same as above
4. Event to fire: `Lead` (standard Meta event for form signups)
5. Triggering: **New Trigger**
   - Type: **Custom Event**
   - Event name: `waitlist_signup` (must match exactly — this is what the form pushes to dataLayer)
   - Name: `Custom Event — waitlist_signup`
6. Name: `Meta Pixel — Lead (Waitlist Signup)`
7. **Save**

### 4c — Publish

**GTM → Submit → Publish**. Name the version `v1 — Meta Pixel + Lead Event`.

---

## Step 5 — Verify Everything is Firing

### GTM Tag Assistant
1. GTM → **Preview** → enter `https://stylestack-demo.pages.dev`
2. Tag Assistant connects — you should see `Meta Pixel — Page View` under **Tags Fired** on page load
3. Submit the waitlist form on the demo site
4. In Tag Assistant → click the **waitlist_signup** custom event on the left
5. Confirm `Meta Pixel — Lead (Waitlist Signup)` appears under **Tags Fired**

### Meta Events Manager
1. Go to [business.facebook.com/events_manager](https://business.facebook.com/events_manager)
2. Open your StyleStack pixel
3. Use the **Test Events** tab — enter your site URL and submit the form
4. You should see a `Lead` event appear in real time (green checkmark)

---

## Step 6 — Take Screenshots

Save all screenshots to `screenshots/` in this folder (alongside the Stellapura ones):

| Screenshot | What to capture |
|---|---|
| `gtm_stylestack_overview.png` | GTM workspace showing both tags (Page View + Lead) |
| `gtm_stylestack_tag_assist.png` | Tag Assistant showing `waitlist_signup` event with Meta Pixel Lead tag fired |
| `meta_pixel_page_view.png` | Events Manager → Test Events showing PageView |
| `meta_pixel_lead_event.png` | Events Manager → Test Events showing Lead event from form submission |

---

## Step 7 — Update CASE_STUDY.md

Add a "Case Study 2" section to `CASE_STUDY.md` (or create `CASE_STUDY_02_STYLESTACK.md`) documenting:
- What was built (Meta Pixel via GTM on Cloudflare Pages static site)
- The conversion event approach (dataLayer.push → GTM Custom Event trigger → Meta Pixel Lead tag)
- How it closes the gap noted in Case Study 1

---

## Summary of What This Adds to the Portfolio

| Skill | Evidence |
|---|---|
| Meta Pixel installation | Pixel base code delivered via GTM, not direct script install |
| GTM conversion event via dataLayer | `waitlist_signup` custom event fires before GTM trigger |
| Multi-platform tag management | Two tags (PageView + Lead) managed in one GTM container, no code changes needed to add more |
| Cloudflare Pages static deployment | Second live site, separate from the Stellapura Workers project |
| Clean container hygiene | New dedicated container, not cross-contaminating Stellapura's data |
