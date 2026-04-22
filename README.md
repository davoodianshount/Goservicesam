# goservicesam.com

Website for **Service Sam Electric** — Licensed C10 Electrical Contractor in Los Angeles.

- **Business:** Service Sam Electric
- **Owner:** Sam Davoodian
- **License:** CSLB C10 #1125961
- **Phone:** (818) 934-2824
- **Email:** Sam@goservicesam.com
- **Address:** 121 W Lexington Dr, Suite V100, Glendale, CA 91203

---

## File structure

```
/
├── index.html          ← main site (single file, all pages as hash-routed sections)
├── 404.html            ← branded not-found page
├── robots.txt
├── sitemap.xml
├── _redirects          ← Cloudflare Pages redirect rules (www → apex, http → https)
└── assets/
    ├── favicon.svg
    ├── apple-touch-icon.png
    ├── icon-512.png
    └── og-image.png    ← social share image (1200×630)
```

---

## Before you deploy: one thing to swap

The quote form posts to [Web3Forms](https://web3forms.com). You need an access key tied to `Sam@goservicesam.com`:

1. Go to https://web3forms.com → paste `Sam@goservicesam.com` → they email a key immediately.
2. Open `index.html`, find `YOUR_SERVICE_SAM_WEB3FORMS_KEY` (one occurrence, around line 816), paste the key in.
3. Commit + push — form submissions now land in Sam's inbox.

If you want to test the pipeline immediately without creating a new key, you can temporarily use your Rate Hero key (`544fd03b-53dd-4844-ae11-af8c8871adf8`) — submissions will go to **your** registered email, not Sam's. Swap it for Sam's real key before launch.

---

## Deployment: GitHub → Cloudflare Pages → Domain

### 1. Create the GitHub repo

```bash
cd goservicesam
git init
git add .
git commit -m "Initial site"
gh repo create davoodianshount/Goservicesam --public --source=. --remote=origin --push
```

Or via the web: create a new empty repo `davoodianshount/Goservicesam`, then:
```bash
git remote add origin https://github.com/davoodianshount/Goservicesam.git
git branch -M main
git push -u origin main
```

### 2. Connect to Cloudflare Pages

1. Log into Cloudflare → **Workers & Pages** → **Create application** → **Pages** → **Connect to Git**.
2. Authorize GitHub, pick `davoodianshount/Goservicesam`.
3. Framework preset: **None**.
4. Build command: *leave blank*
5. Build output directory: `/` (root)
6. Save and deploy.

Within 60 seconds you'll get a `goservicesam.pages.dev` URL. Verify the site loads correctly there before pointing the domain.

### 3. Point goservicesam.com at Cloudflare

Since Google Domains was migrated to Squarespace in 2023, your domain is now managed in **Squarespace Domains** (log in with your Google account at https://account.squarespace.com/domains).

**The cleanest path is to transfer DNS management to Cloudflare** (Cloudflare becomes your nameserver; domain ownership stays with Squarespace):

1. In Cloudflare → **Websites** → **Add site** → `goservicesam.com`. Pick the **Free** plan.
2. Cloudflare scans existing DNS and gives you **two nameservers** (e.g., `xxx.ns.cloudflare.com`).
3. In Squarespace Domains → goservicesam.com → **DNS Settings** → **Custom Nameservers**. Paste the two Cloudflare nameservers, remove the Squarespace defaults, save.
4. Propagation takes 5 minutes to a few hours.
5. Once Cloudflare shows the domain as "Active":
   - Cloudflare → **Workers & Pages** → your `Goservicesam` project → **Custom domains** → **Set up a custom domain** → `goservicesam.com`. Add again for `www.goservicesam.com`.
   - Cloudflare creates the CNAME records automatically.
6. SSL: Cloudflare → SSL/TLS → **Full (strict)**. Cert provisions automatically (usually minutes).

**If you want to keep DNS at Squarespace** (skip nameserver changes): add these records in Squarespace DNS instead:
- `CNAME  @    goservicesam.pages.dev` (only works if Squarespace supports CNAME flattening on apex; otherwise use their A-record ALIAS feature, or switch to Cloudflare nameservers above)
- `CNAME  www  goservicesam.pages.dev`

The nameserver transfer is cleaner — recommended.

### 4. Post-launch checklist

- [ ] Verify site loads at `https://goservicesam.com` (not just `pages.dev`)
- [ ] Test `www.goservicesam.com` redirects to apex
- [ ] Submit a test lead through the form → confirm it arrives at Sam's inbox
- [ ] [Google Search Console](https://search.google.com/search-console) → add property → verify ownership (DNS TXT record method works well with Cloudflare). Submit `https://goservicesam.com/sitemap.xml`.
- [ ] Claim [Google Business Profile](https://business.google.com/) — this is the single biggest lever for local electrician SEO. Use the exact NAP (Name, Address, Phone) that matches the site:
  - Name: Service Sam Electric
  - Address: 121 W Lexington Dr, Suite V100, Glendale, CA 91203
  - Phone: (818) 934-2824
  - Category: Electrician (primary), Electrical repair shop (secondary)
  - License: C10 #1125961
- [ ] Add the business to [Yelp](https://biz.yelp.com/), [Nextdoor Business](https://business.nextdoor.com/), [BBB](https://www.bbb.org/), [Angi](https://www.angi.com/) — consistent NAP across all.
- [ ] Set up a Google Voice or CallRail number if you want to track which calls come from the website vs. other sources.

---

## Known limitations / future work

- **Service sub-pages use hash routing** (`#emergency`, `#panel`, etc.) rather than real URLs. This is fine for UX but isn't ideal for SEO — Google indexes each hash URL as the root page. If organic traffic becomes a priority, convert these to real routes (`/services/emergency.html`, etc.) so each has its own meta/title/H1/schema.
- **Reviews are hardcoded** — the "4.9 on Google, 523 reviews" is the design value. Once the real Google Business Profile has reviews, decide whether to hardcode the current count or pull via an API.
- **Brand logo is inlined as a base64 PNG** in `index.html` (~100KB). Low priority to fix, but eventually worth extracting to `/assets/logo.png`.

---

## Brand reference

- Navy: `#0B1E3F`
- Yellow: `#FFC422`
- Blue accent: `#1E5FD4` / `#2f74ef`
- Fonts: Archivo Black (display), Inter (body)
