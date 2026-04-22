# Service Sam Electric — Site Upgrade Package

This package contains three PR-ready changes for `goservicesam.com`:

1. **Schema bug fixes** on the homepage
2. **Hash routes rewritten as real routes** (`/services/*`)
3. **Photo-upload lead magnet** at `/safety-check`

Overall line-level change: **+~2,500 lines added, ~17KB + 20KB removed from `index.html`** (no net bloat — the homepage actually got lighter).

---

## File manifest

```
index.html                              ← edited (schema + nav + legacy redirect; -39KB)
sitemap.xml                             ← replaced (9 URLs instead of 1)
safety-check.html                       ← NEW — photo upload Zinsco/FPE LP
assets/site.css                         ← NEW — shared stylesheet for sub-pages
services/
  ├── emergency-electrical.html         ← NEW
  ├── panel-upgrade.html                ← NEW
  ├── ev-charger-installation.html      ← NEW
  ├── whole-home-rewiring.html          ← NEW
  ├── troubleshooting.html              ← NEW
  ├── lighting-installation.html        ← NEW
  └── commercial-electrical.html        ← NEW
01-schema-fix.patch                     ← standalone unified diff of the schema-only fix
_generate_service_pages.py              ← Python generator so content can be regenerated
```

Untouched: `404.html`, `robots.txt`, `wrangler.toml`, `assets/favicon.svg`, `assets/apple-touch-icon.png`, `assets/icon-512.png`, `assets/og-image.png`.

---

## 1 · Schema bug fixes (`index.html`)

### What changed

| Before | After |
| --- | --- |
| `"image": "data:image/png;base64,…"` (20KB data URI) | `"image": "https://goservicesam.com/assets/og-image.png"` + `"logo":` URL |
| `"aggregateRating": { "ratingValue": "5.0", "reviewCount": "10" }` | `reviewCount: "3"` — **matches the 3 reviews actually on the page** |
| No `sameAs` | Deep links to Google / Yelp / HomeAdvisor |
| No `geo` | Glendale coordinates |
| No `@id` | `@id: "https://goservicesam.com/#business"` so sub-pages can `"provider": {"@id": …}` |
| No offer catalog | `hasOfferCatalog` with all 7 services linked |
| Inconsistent hours format | Both `openingHours` (short) and `openingHoursSpecification` (structured) |

### Why each matters

- **Base64 image**: Google's structured-data parser silently drops `data:` URIs. The logo never appeared in rich results. Fixed.
- **Inflated `reviewCount`**: this is the most important fix. Claiming 10 reviews when 3 are on the page violates Google's [review snippet policy](https://developers.google.com/search/docs/appearance/structured-data/review-snippet) and can trigger a manual action that removes rich results **sitewide**. `reviewCount: "3"` is conservative but safe. **When the Google Business Profile has more real reviews, bump this number to match the GBP count — must not exceed what's visible on the page or on the linked profile.**
- **`sameAs`**: Google uses this for entity resolution — connecting the website, the GBP, the Yelp profile, and the HomeAdvisor profile as the same real-world business. Strengthens Knowledge Panel.
- **`@id` anchor**: each `/services/*` page's Service schema does `"provider": {"@id": "https://goservicesam.com/#business"}` instead of repeating the whole business block — cleaner and lets Google confidently attribute each service to the same entity.
- **`hasOfferCatalog`**: discovery signal. When Google crawls the homepage, it now sees all 7 services linked as `Offer → Service` with their canonical URLs.

### Standalone diff

`01-schema-fix.patch` is a clean unified diff of ONLY the schema changes. Applicable as `git apply 01-schema-fix.patch` if you want to ship the schema fix independently of the route changes. (The full patched `index.html` in this package already has it applied.)

---

## 2 · Hash routes → real routes

### What changed

**Before:** Service sub-pages existed as `#emergency`, `#panel`, `#ev`, etc. — hash fragments on the homepage, rendered via the `:target` CSS trick. Google treated every hash URL as the root page.

**After:** Each is now a real URL with its own meta, H1, canonical, OG tags, Service schema, BreadcrumbList schema, and FAQPage schema.

| Old URL | New URL |
| --- | --- |
| `/#emergency` | `/services/emergency-electrical` |
| `/#panel` | `/services/panel-upgrade` |
| `/#ev` | `/services/ev-charger-installation` |
| `/#rewire` | `/services/whole-home-rewiring` |
| `/#troubleshoot` | `/services/troubleshooting` |
| `/#lighting` | `/services/lighting-installation` |
| `/#commercial` | `/services/commercial-electrical` |

### Each service page includes

- Unique `<title>` (~60–70 chars, keyword + "in Los Angeles" + brand)
- Unique meta description (~155 chars)
- Canonical URL
- Own OG + Twitter card
- **Service** schema with `provider: {@id}` → main business, `areaServed` array of LA-area cities, `offers` with starting price
- **BreadcrumbList** schema
- **FAQPage** schema (4 Q&As per page — this is the single biggest rich-result win)
- H1 matching the search query intent
- Price band (starting-from price + typical timeline)
- "What we handle" checks list
- "Benefits / process" checks list
- Inline FAQ with `<details>` elements mirroring the FAQPage schema
- CTA block with the same Call / Text / Estimate three-button pattern
- Footer with license # and cross-links to other services
- Mobile sticky CTA

### Backward compatibility

A tiny JS snippet at the top of `index.html` (line ~759) catches any legacy `/#emergency`, `/#panel`, etc. URLs and `location.replace()`-es them to the new routes. Bookmarks and shared links keep working. Not a server redirect — hashes never reach the server — but good enough for humans. Google will re-index via the new sitemap.

### Cloudflare Pages behavior

Cloudflare Pages' `not_found_handling = "404-page"` in `wrangler.toml` serves files at both `/services/panel-upgrade.html` and `/services/panel-upgrade` (trailing-slash-less, extension-less). The canonical URLs use the extension-less form, which is the Google-preferred convention.

### What's NOT converted

The **city pages** (`#la`, `#glendale`, `#burbank`, etc.) are still hash-routed in `index.html`. Scope was the 7 services only. Converting city pages is the obvious next project — same pattern, same template, pulls the same shared CSS.

---

## 3 · Photo-upload safety check (`/safety-check`)

### The angle

A free lead magnet specifically targeting homes with old panels. User uploads photos of their electrical panel, you reply within 24 hours telling them whether it's a Federal Pacific, Zinsco, Challenger, or safe brand. If it's one of the dangerous brands, it funnels naturally into a panel-upgrade quote.

### Form fields

| Field | Required | Notes |
| --- | --- | --- |
| Name | ✓ | |
| Phone | ✓ | Used for priority follow-up when we spot FPE/Zinsco |
| Email | ✓ | Where the written reply goes |
| Property address | ○ | Helps crew routing if they book a visit |
| Year built | ○ | Low-commitment qualifying signal |
| Panel photos | ✓ | Up to 4, max 5MB each |
| Additional notes | ○ | |

### Submission pipeline

`<form enctype="multipart/form-data">` → POST to `https://api.web3forms.com/submit` with the same access key you're already using on the main quote form. Files arrive as email attachments in Sam's inbox. Same pipeline you've got running for the Rate Hero leads — no new infrastructure.

**Web3Forms attachment limits to verify on Sam's plan:**
- Free plan: attachment support is typically capped at ~5MB total per submission
- Hobby plan ($5/mo): 10MB per submission
- If modern phone photos (often 3–5MB each portrait-mode) push past the limit, either (a) upgrade the Web3Forms plan, (b) swap to Formspree (25MB per submission on free tier), or (c) pipe through Cloudflare R2 for object storage. The client-side validator warns the user before they hit submit, so they won't hit a failed submission blind.

The submit handler shows a success state inline (no redirect) and clears the form. On error, the submit button turns red with a "call us directly" fallback — never a broken silent fail.

### Schema

- `Service` schema with `offers: { price: "0" }` (explicitly labels it free — good for search)
- `BreadcrumbList`
- `FAQPage` with 5 Q&As addressing the most common homeowner concerns (is this really free, what do you do with my photos, etc.)

### UX notes worth testing

- `capture="environment"` on the file input triggers the rear camera on mobile — nice one-tap flow ("take photo of panel")
- Safety note box explicitly says "do not remove any covers" — avoids liability from DIY electricians pulling dead-fronts
- Success state auto-fires a `dataLayer.push({event: 'safety_check_submit'})` — wire a GTM container if you want this to show up as a conversion in GA4 / Meta Ads / Google Ads

---

## Pre-deploy checklist

### Required before merge

- [ ] **Schema review**: confirm `reviewCount: "3"` is the number you want publicly. If the real Google Business Profile has more real reviews, bump this — but it must match what's visible on the page AND linked GBP.
- [ ] **Web3Forms attachment limit**: confirm Sam's current plan supports attachments at the size you expect. Test the safety-check form with a typical phone photo before sharing the URL anywhere.
- [ ] **Visual spot-check** of each of the 7 service pages at both mobile (430px) and desktop (1100px+) widths.
- [ ] **Test legacy hash redirects**: visit `https://goservicesam.com/#panel` — should immediately land on `/services/panel-upgrade`.

### Post-deploy

- [ ] **Google Search Console**: submit the new `sitemap.xml`. Then use the URL Inspection tool on each of the 8 new URLs and click "Request Indexing" — this speeds up crawling from days to hours.
- [ ] **Google Rich Results Test**: paste each of the 8 URLs into [search.google.com/test/rich-results](https://search.google.com/test/rich-results) and confirm no errors. Each service page should light up FAQ, Breadcrumbs, and Service snippets.
- [ ] **Schema.org validator**: [validator.schema.org](https://validator.schema.org/) — paste each URL, confirm zero errors.
- [ ] **Meta tags preview**: paste each URL into [metatags.io](https://metatags.io) to confirm OG previews look right on iMessage, Facebook, Slack, and Twitter.
- [ ] **CWV check**: [pagespeed.web.dev](https://pagespeed.web.dev/) for each new route. Should be 90+ across the board since pages are static HTML + one small CSS file.
- [ ] **Test form submission**: submit a test safety-check from your phone with a real photo. Confirm it arrives in Sam's inbox with the attachment.
- [ ] **Add the safety-check URL to Google Ads and direct-mail campaigns.** It's the highest-leverage URL on the site.

### Marketing plays this unlocks

- The `/safety-check` page is the killer-hook version of a panel-upgrade Google Ads campaign. Bid on `"is my panel safe"`, `"zinsco panel"`, `"fpe panel"`, `"federal pacific fire"` etc. Free photo-based answer converts ridiculously well against quote-gating competitors.
- Panel-upgrade and EV-charger service pages can be used as standalone landing pages for Meta Ads and Nextdoor sponsored posts. They're fully self-contained with their own CTAs.
- `/safety-check` works great on direct mail. Print QR code on a Zinsco-targeted mailer → scans go straight to the upload form.
- `goservicesam.com/safety-check` is short and tells a real-estate agent you're serious when they ask "what do you do for old homes?" at a realtor meetup.

---

## Known limitations / scope boundaries

- **City pages** (`/#glendale`, `/#burbank`, etc.) were NOT converted to real routes — out of scope. Next project. Template can be adapted from `/services/*.html`.
- **Inline logo base64 in index.html**: the schema `image` URL is fixed, but the `<img>` logo in the topbar is still inline base64 (~100KB). Moving it to `/assets/logo.png` would shave another 100KB off the homepage. Left untouched to keep this patch focused.
- **No analytics tags wired in** — the safety-check form fires a `dataLayer.push` on success but there's no GTM container on the site yet. 10-min add when you're ready.
- **Form attachment size validation is client-side only.** A determined user can still submit a 50MB file through DevTools. Web3Forms will reject it server-side. Not a real issue, but noting for completeness.
- **No About Sam page / no project gallery / no Google Maps embed** — all three were in the original audit as high-leverage additions but are separate scoped work items.

---

## Addendum: Homepage surfacing of `/safety-check`

After the initial package, the safety-check URL was added in **three** places on the site so it's genuinely discoverable:

1. **Above-the-fold dashed link** under the hero stats: *"Not sure about your panel? Free photo check →"*. Visible on first render on any screen size.
2. **Yellow callout band** between the Services grid and the Area section — hard to miss, uses the brand yellow for "free/urgent" signal without competing with the main navy hero.
3. **Footer legal line** (already there) — long-tail discoverability for users who scroll all the way.

If you want a fourth touchpoint, adding the safety-check link as a secondary CTA on `/services/panel-upgrade` (right under the hero price band) is the highest-leverage remaining placement — people who clicked through to the panel page may not yet be ready to quote but are one step from uploading a photo. Ten-minute addition; say the word.

---

## Quick rollback

If anything goes sideways post-deploy:

```bash
git revert <commit-sha>
git push
```

Cloudflare Pages redeploys in under 60 seconds.

The **only** change with non-trivial rollback complexity is the schema — if you've already submitted the new URLs to Google Search Console and want to roll back the service pages, Google will see 404s and deindex them over a few days. Not a crisis, just noting.

---

## Brand constants used (unchanged from current site)

- Navy: `#0B1E3F`, `#122a54`, `#07152e`
- Yellow: `#FFC422`, `#ffb300`
- Blue: `#1E5FD4`, `#2f74ef`
- Fonts: **Archivo Black** (display), **Inter** (body)
- Web3Forms access key: `11ac3944-7c91-4632-9b00-2c6071c85345` (Sam's — confirmed already swapped out from the Rate Hero key)

Business NAP (canonical across all schema blocks):
- **Service Sam Electric**
- 121 W Lexington Dr, Suite V100, Glendale, CA 91203
- (818) 934-2824
- Sam@goservicesam.com
- CA C10 License #1125961
