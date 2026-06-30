# GTM + GA4 Conversion Tracking — Build Instructions

Step-by-step record of what was built and how. Replicable on any server-rendered site.

---

## Prerequisites

- Google account
- A live website you can modify the source code of
- For Cloudflare Workers sites: `wrangler` CLI access

---

## Step 1 — Create GTM Account and Container

1. Go to [tagmanager.google.com](https://tagmanager.google.com)
2. Click **Create Account**
3. Account name: your project name (e.g. `Marcus Portfolio`)
4. Container name: your site domain (e.g. `stellapura.maillab.live`)
5. Target platform: **Web**
6. Click **Create** → accept terms
7. Note the Container ID (format: `GTM-XXXXXXX`)

---

## Step 2 — Install GTM on the Site

GTM requires two snippets: one in `<head>`, one immediately after `<body>`.

### For standard HTML sites
Paste exactly as shown in the GTM install popup.

### For Hono JSX (Cloudflare Workers)
JSX escapes string content by default — pasting the snippet as a template literal breaks the JavaScript. Use `dangerouslySetInnerHTML` instead:

In your shared layout file (e.g. `src/views/public-layout.tsx`):

```tsx
// In <head> — first element:
<script dangerouslySetInnerHTML={{ __html: "(function(w,d,s,l,i){w[l]=w[l]||[];w[l].push({'gtm.start':new Date().getTime(),event:'gtm.js'});var f=d.getElementsByTagName(s)[0],j=d.createElement(s),dl=l!='dataLayer'?'&l='+l:'';j.async=true;j.src='https://www.googletagmanager.com/gtm.js?id='+i+dl;f.parentNode.insertBefore(j,f);})(window,document,'script','dataLayer','GTM-XXXXXXX');" }} />

// Immediately after <body>:
<noscript>{`<iframe src="https://www.googletagmanager.com/ns.html?id=GTM-XXXXXXX" height="0" width="0" style="display:none;visibility:hidden"></iframe>`}</noscript>
```

Replace `GTM-XXXXXXX` with your actual container ID.

If you have multiple layout files (e.g. admin layout + public layout), add the snippet to **both**.

### Deploy
```powershell
npx wrangler deploy
```

---

## Step 3 — Verify GTM is Firing

1. In GTM → click **Preview** (top right)
2. Enter your site URL → click **Connect**
3. A Tag Assistant banner appears at the bottom of your site: **"Tag Assistant Connected"**
4. If not connected: check that the snippet deployed correctly by viewing page source and searching for your Container ID

---

## Step 4 — Create GA4 Property

1. Go to [analytics.google.com](https://analytics.google.com)
2. **Admin** → **Create Property**
3. Property name: your project name
4. Platform: **Web**
5. Enter your site URL
6. Copy the **Measurement ID** (format: `G-XXXXXXXXXX`)

---

## Step 5 — Add GA4 Configuration Tag in GTM

1. GTM → **Tags** → **New**
2. Tag Configuration → **Google Tag**
3. Tag ID: your Measurement ID (`G-XXXXXXXXXX`)
4. Triggering → **All Pages**
5. Name: `GA4 - Configuration`
6. **Save**

---

## Step 6 — Add Conversion Event Tag

1. GTM → **Tags** → **New**
2. Tag Configuration → **Google Analytics: GA4 Event**
3. Measurement ID: `G-XXXXXXXXXX`
4. Event name: `agent_registration` (or whatever conversion you're tracking)
5. **Triggering** → **New Trigger**
   - Type: **Form Submission**
   - Fire on: **Some Forms**
   - Condition: Page URL → contains → `/register` (scope to the specific page)
   - Name: `Form Submit - Register`
6. Name the tag: `GA4 - Agent Registration`
7. **Save**

---

## Step 7 — Publish GTM Container

1. GTM → **Submit** (top right)
2. Version name: `v1` or `Initial setup`
3. Click **Publish**

All saved tags go live with the published version. GTM requires an explicit publish step — saved tags do not fire until published.

---

## Step 8 — Verify Conversion Tracking

**GA4 Realtime:**
1. Go to GA4 → **Reports** → **Realtime**
2. Open your site in another tab and navigate around
3. You should see active users appear within 30 seconds

**Conversion event test:**
1. GTM → **Preview** → connect your site
2. Navigate to the form page and submit the form
3. In the Tag Assistant debug panel → click **Form Submit** event on the left
4. Confirm your event tag appears under **Tags Fired**

---

## Result

- GTM container managing all tags from one dashboard
- GA4 receiving page view data across all pages
- Custom conversion event firing on form submission
- No code changes needed to add future tags

---

## Known Gaps / Next Steps

- **Google Ads conversion tag** — import GA4 conversion into Google Ads for ROAS reporting
- **Meta Pixel** — add via GTM using same pattern (Meta Pixel tag type + event trigger)
- **Consent mode** — required for EU traffic; add CMP integration before running paid ads
