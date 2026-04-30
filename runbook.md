# MyFuture — Deployment & Security Runbook

A practical, step-by-step guide to taking the MyFuture site from local files to a properly secure live deployment. Aimed at completion in a focused weekend.

---

## What's in this folder

| File | Purpose | Where it goes when deployed |
|---|---|---|
| `_headers` | Security headers (CSP, HSTS, etc.) for Cloudflare Pages / Netlify | Repo root |
| `robots.txt` | Search engine crawler directives | Repo root → served at `/robots.txt` |
| `sitemap.xml` | Site map for search indexing | Repo root → served at `/sitemap.xml` |
| `privacy.html` | Starter privacy policy (DRAFT — needs legal review) | Repo root → served at `/privacy` |
| `runbook.md` | This file | Keep in repo, do not deploy |

You will also need the actual site HTML (`myfuture-site.html` and `myfuture-investor-site.html` from the outputs folder), which become `index.html` and `investors.html` in your repo.

---

## Phase 1 — Repo & domain (30 minutes)

### 1. Create the repo

```bash
# In a new local folder
mkdir myfuture-web && cd myfuture-web
git init

# Add files
cp /path/to/myfuture-site.html ./index.html
cp /path/to/myfuture-investor-site.html ./investors.html
cp /path/to/deployment/_headers .
cp /path/to/deployment/robots.txt .
cp /path/to/deployment/sitemap.xml .
cp /path/to/deployment/privacy.html ./privacy.html

# Add a .gitignore
cat > .gitignore <<EOF
.DS_Store
node_modules/
.env
.env.*
*.log
.vscode/
.idea/
EOF

# Commit
git add .
git commit -m "Initial site"
```

Push to GitHub (private repo). You can use GitHub's web UI or `gh repo create myfuture-web --private --source=. --push`.

### 2. Buy the domain

- Go to **Cloudflare Registrar** (`dash.cloudflare.com → Domain Registration → Register Domains`)
- Search for `myfuture.africa` (or whatever you've decided)
- Cloudflare sells at cost — no markup, free WHOIS privacy
- Pay for at least 1 year. 5+ years signals legitimacy to Google.

### 3. Enable two-factor authentication everywhere

Before you do anything else:

- Cloudflare account → 2FA on (use an authenticator app, not SMS)
- GitHub account → 2FA on
- Your email account → 2FA on
- Generate and securely store recovery codes for all three

This is the single most important security control you will set up. Most "hacks" are credential takeovers, not zero-days.

---

## Phase 2 — Cloudflare Pages deployment (20 minutes)

### 1. Connect the repo

- Cloudflare dashboard → **Workers & Pages** → **Create application** → **Pages** → **Connect to Git**
- Authorise Cloudflare to access your GitHub
- Select the `myfuture-web` repo
- Build settings:
  - **Framework preset:** None
  - **Build command:** (leave empty)
  - **Build output directory:** `/`
- Click **Save and Deploy**

In about 60 seconds you'll have a live URL like `myfuture-web.pages.dev`. Open it and check the site renders.

### 2. Add your custom domain

- In your Pages project → **Custom domains** → **Set up a custom domain**
- Enter `myfuture.africa` and `www.myfuture.africa` (do both)
- Cloudflare auto-creates the DNS records since you bought the domain through them
- HTTPS certificate issues automatically within ~5 minutes

### 3. Verify the security headers are live

Once the site is live at your real domain:

- Open `https://securityheaders.com/?q=myfuture.africa`
- You should get an **A or A+ grade**
- If anything's red, check the `_headers` file syntax (Cloudflare Pages is sensitive to indentation — two spaces, not tabs)

Also test:

- `https://www.ssllabs.com/ssltest/analyze.html?d=myfuture.africa` — aim for A+
- Visit `http://myfuture.africa` (no HTTPS) — should redirect to HTTPS automatically

---

## Phase 3 — Cloudflare security configuration (15 minutes)

In the Cloudflare dashboard for your domain:

### SSL/TLS settings

- **Overview** → SSL/TLS encryption mode: **Full (strict)**
- **Edge Certificates** → Always Use HTTPS: **On**
- **Edge Certificates** → Minimum TLS Version: **TLS 1.2**
- **Edge Certificates** → TLS 1.3: **On**
- **Edge Certificates** → Automatic HTTPS Rewrites: **On**

### Security settings

- **Security** → **Bots** → **Bot Fight Mode**: **On** (free)
- **Security** → **DDoS** → leave on the default settings (already active)
- **Security** → **Settings** → Security Level: **Medium**
- **Security** → **Settings** → Browser Integrity Check: **On**

### Rate limiting (free tier allows one rule)

- **Security** → **WAF** → **Rate limiting rules** → **Create rule**
- Name: `Form spam protection`
- If incoming requests match: URI Path contains `/form` or `/submit`
- Then: Block, for 10 minutes, when more than 10 requests in 1 minute from the same IP

### Page Rules

- **Rules** → **Page Rules** → **Create Page Rule**
- URL: `*myfuture.africa/admin*` → Settings: Cache Level = Bypass, Disable Apps
- URL: `*myfuture.africa/.git*` → Forwarding URL → 404 (catches anyone scanning for leaked Git data)

---

## Phase 4 — Forms & lead capture (45 minutes)

Your site has CTA forms (assessment signup, demo booking, partner enquiry). Don't roll your own backend yet.

### Use Formspree + Cloudflare Turnstile

1. Sign up at `formspree.io` (free tier is fine to start). 2FA on the account.
2. Create a form for each CTA. Note the form endpoint URL (looks like `https://formspree.io/f/xxxxxx`).
3. Sign up for **Cloudflare Turnstile** (free, privacy-respecting CAPTCHA alternative). Get a site key.
4. In your `index.html`, replace each CTA's `<a href="#">` with a real form:

```html
<form action="https://formspree.io/f/YOUR_FORM_ID" method="POST">
  <input type="email" name="email" required placeholder="Your email">
  <input type="hidden" name="_subject" value="New MyFuture signup">
  <!-- Cloudflare Turnstile widget -->
  <div class="cf-turnstile" data-sitekey="YOUR_TURNSTILE_SITE_KEY"></div>
  <button type="submit">Get my report</button>
</form>
<script src="https://challenges.cloudflare.com/turnstile/v0/api.js" defer></script>
```

You'll need to add `https://challenges.cloudflare.com` to your CSP `script-src` and `frame-src` directives in `_headers` for this to work.

### What never goes in your HTML

- API keys, secret keys, access tokens of any kind
- Database credentials
- Internal URLs, debug endpoints
- Anything you wouldn't put on a billboard

If you ever need a server-side secret, use Cloudflare Workers or Pages Functions and put the secret in environment variables.

---

## Phase 5 — Privacy, analytics, and monitoring (30 minutes)

### 1. Privacy & terms

- Open `privacy.html` and fill in every `[placeholder]` — entity name, address, Information Officer name, registration number, processor list
- Send it to a privacy lawyer for review. Cost in SA: typically R5,000–R15,000 for review and revision. Worth every cent before you collect a single email.
- Register your **Information Officer** with the SA Information Regulator at inforegulator.org.za. This is a legal requirement before processing personal information at scale.
- Draft a `terms.html` (terms of use) — your lawyer can do both at the same time.

### 2. Privacy-first analytics

Skip Google Analytics. For an audience that includes minors, use:

- **Plausible** (`plausible.io`) — €9/month, no cookies, GDPR/POPIA-compliant by default
- **Fathom** (`usefathom.com`) — similar, slightly more expensive
- **Cloudflare Web Analytics** — free, basic, but built in

Paste the analytics snippet into your HTML `<head>`. Add the analytics domain to your CSP `script-src`.

### 3. Cookie banner

Install **Cloudflare Zaraz** (free) for consent management, or use **Cookiebot** (free for low traffic).

If you only use strictly necessary cookies, you don't legally need a banner — but a clear "We respect your privacy" notice still helps build trust.

### 4. Uptime monitoring

- **UptimeRobot** (free) — ping every 5 minutes, alert by email if down
- Set up a check for `https://myfuture.africa` and `https://myfuture.africa/privacy`

### 5. Email security

If you'll send emails from `@myfuture.africa`:

- Set up **SPF, DKIM, and DMARC** records — Cloudflare's Email Security tool guides you
- Use a real email provider (Google Workspace, Fastmail) for `hello@`, `privacy@`, `security@`, `partners@`
- Don't ever send from a free Gmail address with a forwarding rule — looks unprofessional and fails spam filters

---

## Phase 6 — Launch checklist

Run through this before you tell anyone the URL exists.

### Content

- [ ] Every `[placeholder]` in privacy.html replaced
- [ ] Privacy policy reviewed by a lawyer
- [ ] Terms of use drafted, reviewed, published
- [ ] Cookie policy or notice published
- [ ] Information Officer registered with the SA Information Regulator
- [ ] All `#` placeholder links in the site HTML replaced with real URLs or removed
- [ ] All form actions point to real endpoints (Formspree, etc.)
- [ ] WhatsApp number `27000000000` replaced with the real counsellor line
- [ ] Pilot stats (12,400 / 38 / 3.4× / 62% / 94%) replaced with real or honest "early-stage" numbers
- [ ] Partner names verified (or section removed if no signed partners yet)
- [ ] Testimonial photos consented in writing

### Security

- [ ] securityheaders.com grade: A or A+
- [ ] ssllabs.com grade: A or A+
- [ ] HTTP redirects to HTTPS
- [ ] HSTS header sent
- [ ] CSP tested without `'unsafe-inline'` (long-term goal)
- [ ] 2FA enabled on Cloudflare, GitHub, Formspree, email
- [ ] Domain registrar lock enabled
- [ ] No secrets in the public repo (`git log -p | grep -iE "key|secret|token|password"` returns nothing useful)
- [ ] `.env` files in `.gitignore`
- [ ] Bot Fight Mode on
- [ ] Rate limiting active on form endpoints

### SEO & analytics

- [ ] Sitemap submitted to Google Search Console
- [ ] Sitemap submitted to Bing Webmaster Tools
- [ ] robots.txt accessible at `/robots.txt`
- [ ] Open Graph tags in HTML for social sharing
- [ ] Favicon present
- [ ] Page titles and meta descriptions on every page

### Resilience

- [ ] Uptime monitoring configured
- [ ] Email alerts going to a monitored address (not your daily inbox)
- [ ] Backup plan: Pages auto-redeploys from Git, so the repo is your backup. Make sure it's pushed.

---

## When to graduate to a real backend

Stay on a static site for as long as you possibly can. The moment you need:

- Real user accounts with stored passwords
- A database of learner profiles
- The actual aptitude assessment engine
- Payment processing

…that's a full application, not a website, and it deserves a separate security architecture document, a threat model, and (given child data) a privacy impact assessment before a single user signs up.

At that point talk to:

- A security architect for the application design
- An external pen-test firm for pre-launch testing
- Your lawyer to update the privacy policy for the new processing activities

---

## What to do if something goes wrong

### Suspected breach

1. Don't panic. Don't delete anything (preserves evidence).
2. Email `security@myfuture.africa` (set up an inbox for this) with the details.
3. Within 72 hours, assess scope, affected users, and notification obligations under POPIA section 22.
4. Notify the Information Regulator and affected users where required.

### Site defaced or compromised

1. Cloudflare → **Pause Cloudflare on Site** (under Overview) — takes the site offline.
2. Roll back to the last known-good Git commit. Force redeploy.
3. Rotate all credentials (Cloudflare, GitHub, Formspree, email).
4. Investigate how access was gained before bringing the site back up.

### DDoS attack

You're behind Cloudflare, so this is largely handled automatically. If a sustained attack gets through:

- **Security** → **Settings** → toggle **"I'm Under Attack" mode** (presents a JS challenge to all visitors for 5 seconds)
- Contact Cloudflare support if you're on a paid plan

---

## Useful links

- [Cloudflare Pages docs](https://developers.cloudflare.com/pages/)
- [SA Information Regulator](https://inforegulator.org.za)
- [POPIA, full text](https://popia.co.za)
- [securityheaders.com](https://securityheaders.com)
- [ssllabs.com SSL test](https://www.ssllabs.com/ssltest/)
- [Mozilla Observatory](https://observatory.mozilla.org)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)

---

*Last updated: 30 April 2026*
