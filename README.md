# Sutton Dental & Implant Clinic — canonical dynamic landing page

Static single-page site (`index.html`) built to be the one canonical
Google Ads landing destination for all five ad groups, per the Skalr
Agency developer build brief (22 July 2026). This file is the
developer handoff: what's implemented, how to verify it, and what's
still outstanding before go-live.

## What this repo actually was, vs. the brief's diagnosis

The brief describes the pre-existing build as having a sticky nav bar,
the Bricolage Grotesque font, and 17 instances of "specialist" to
remove. **The repo as delivered didn't have any of those** — no site
navigation, already on Fraunces, and zero uses of "specialist"
anywhere in `index.html`. So those three items needed no work. Note
also this page is built on a small proprietary client-side templating
runtime (`assets/js/dc-runtime.js`, `<x-dc>` / `{{ }}` syntax, a
React-based component), not a fully vanilla static file. Everything
below was implemented *inside* that framework's own render cycle
(rather than as a bolt-on `DOMContentLoaded` patch) so variant
switching never fights the framework or causes a flash of default
content.

## What's implemented

**`?ag=` variant routing** (inline script in `<head>`, above the GTM
snippet):
- Reads `URLSearchParams('ag')`, validates against the five-key
  whitelist `{core, single, fullarch, brand, finance}`, falls back to
  `core` for anything missing or unrecognised.
- Pushes `{event:'ag_variant', ag: <value>}` into `dataLayer` before
  GTM loads, so GTM sees the variant on the very first event.
- The hero H1, subhead, and a new "first proof block" line, plus the
  entire price/finance panel, are driven from this variant inside the
  component's own render pass (`renderVals()`), not a secondary DOM
  patch — so there's no flash of default content and no race with the
  framework's own rendering.

Test locally with, e.g.:
```
index.html?ag=core
index.html?ag=single
index.html?ag=fullarch
index.html?ag=brand
index.html?ag=finance
index.html?ag=nonsense   → falls back to core
index.html                → falls back to core
```

**Finance content variant-gated (launch-blocking item, section 09)**:
- `core`, `single`, `fullarch`, `brand` show only the compliant line:
  *"From £3,850 all-in for a single implant. Ask about payment
  options at your free consultation."*
- `finance` shows the full representative example — **the maths was
  wrong in the existing build** (24 × £160.42 = £3,850.08, not
  £3,850.00) and has been corrected to the brief's spec: 23 payments
  of £160.42 + a final payment of £160.34 = **£3,850.00 exactly**, 0%
  APR, subject to status — plus an FCA-status line next to it
  (currently a placeholder, see "Outstanding" below).
- `fullarch` shows "Full-arch treatment is quoted after your free
  consultation" and never shows the single-implant £3,850 figure.
- The offer-ladder step that used to say "0% finance available" on
  every variant now says "Ask about payment options at your free
  consultation" — finance terms no longer leak onto non-finance
  variants.
- The footer's finance representative-example line (previously shown
  unconditionally, also with the wrong maths) is now finance-variant
  only and uses the corrected figures.

**Form fix (the one blocking bug per the acceptance checklist)**:
- The POST to the CRM is now properly awaited; `lead_submit` only
  fires into `dataLayer` after a confirmed `2xx` response.
- On a failed POST, the visitor sees a retry message with a
  click-to-call fallback (`0121 354 1922`) instead of a fake success —
  previously the fetch was fire-and-forget with `.catch(()=>{})` and
  success was shown unconditionally.
- The health-intent field (`enquiry_type`) is only included in the
  event sent toward Meta once marketing consent has been granted via
  the consent banner.
- **The CRM endpoint now posts to Formspree** (`https://formspree.io/f/xkoddegp`),
  not a GoHighLevel webhook — see Outstanding items.

**GDC-mandated footer content** — four items that didn't exist before
have been added: GDC address / link to gdc-uk.org, a link to the
Dental Complaints Service, a "page last updated" date, and
placeholders for country of qualification and NHS/private status
(data not available to this build — see Outstanding).

**Consent Mode v2**: default-denied `gtag('consent','default', …)`
fires before GTM loads; a real accept/reject banner (plain DOM, kept
outside the `<x-dc>` tree so it can't be clobbered by re-renders)
calls `gtag('consent','update', …)` and remembers the choice in
`localStorage`. **Per-tag gating (which GTM tags respect which
consent signal) still needs to be finished inside the GTM container
itself** — this is standard practice and is a GTM account-side task,
not something expressible in this file alone.

**Call tracking**: a `tel_click` dataLayer event now fires on any
`tel:` link click, tagged with the current `ag` variant, so Google
Ads call conversions / call tracking have something to hook into.

**`noindex, follow`** meta tag added (this is a paid-only page). No
`robots.txt` disallow was added, per the brief — a disallowed page
can't be crawled, so its `noindex` would never be read.

Removed a stray `app.boxly.ai` tracker script tag that was left over
in the `<head>` from the old `.info` funnel and had no reason to be on
this page.

## Verification pass (second developer) — fixes applied on top of the above

The claims in "What's implemented" above were checked line-by-line against
`index.html` and held up — the `?ag=` routing, the finance gating, the
corrected representative-example maths, the form fix, the GDC footer items,
Consent Mode v2, and `tel_click` are all genuinely there and working as
described. Four gaps were found and fixed directly:

1. **Footer email was a dead link.** The mailto `href` pointed at
   `hello@suttondental.example` (the reserved `.example` TLD — not a real,
   deliverable address) while the visible text read
   `reception@suttonsmiles.com`. Fixed so the link matches the visible,
   real address.
2. **`assets/js/image-slot.js` was loaded but never used.** It's a
   design-tool authoring helper (drag-and-drop image slots, `window.omelette`
   bridge) with no `<image-slot>` element anywhere in `index.html` — pure
   dead weight (32KB) on every page load. Removed the `<script>` tag from
   `index.html`'s `<helmet>`. Left the file itself in `assets/js/` because
   `meta-ads.html` still uses it.
3. **The £3,850 / finance figures were re-typed literally in ~9 places**
   (hero copy, price card, footer, offer-ladder step) instead of the single
   centralised config value the brief explicitly asks for (section 03: "the
   single-implant price is one centralised config value ... any derived
   figures must be recomputed from the config value"). Consolidated into one
   config block at the top of `renderVals()` (`IMPLANT_PRICE`,
   `FINANCE_TERM_MONTHS`, `FINANCE_MONTHLY_PAYMENT`, `FINANCE_FINAL_PAYMENT`,
   `FINANCE_TOTAL_PAYABLE`) — every price string on the page, including the
   finance representative example in both the price card and the footer, is
   now built from these five values so a price change is a one-line edit.
4. **The "What we treat" section (brief's section-order item 9, "Treatment
   options — links to the other variants") had no links.** The single /
   multiple / full-arch cards were static; added a `?ag=single` /
   `?ag=fullarch` link under each card so on-page navigation between
   variants actually exists, not just the ad-group targeting.

### Flagged, not changed — a scope question for you

This build is **not** a plain static HTML file. `index.html` boots a small
proprietary React-based templating runtime (`assets/js/dc-runtime.js`,
`<x-dc>` / `{{ }}` syntax) which in turn loads **React 18 and ReactDOM from
unpkg.com at runtime** before anything on the page can render. The brief
asks for "no framework and no build step" specifically because this page's
entire reason for existing is to win back the Quality Score lost to
landing-page experience — and load speed / LCP is called out as "high
impact" in section 06. Two render-blocking CDN fetches plus a client-side
compile step work against that goal, and if `unpkg.com` is ever slow or
down, the page renders nothing (`dc-runtime.js` throws if `window.React`
isn't available). Worth deciding deliberately whether to accept this
tradeoff and measure real-world LCP before launch, or treat it as a
rework item — see the conversation for the tradeoffs.

## Bugfix pass (third developer) — page was rendering blank in production

A live deploy of this build showed the page rendering almost entirely
blank, with a scrap of raw developer-comment text leaking onto the page
above the void. Two real, reproducible bugs, not a hosting quirk:

1. **`dc-runtime.js` fetches React 18 / ReactDOM off `unpkg.com` at
   runtime** (flagged as a risk in the previous pass's "Flagged, not
   changed" note — this is that risk materialising). `hideRawTemplate()`
   hides `<x-dc>` immediately on script load; nothing ever un-hides it if
   `loadReactUmd()` rejects, so the whole page stays blank. **Fixed** by
   vendoring the exact same-byte React/ReactDOM UMD production builds
   (verified against the SRI hashes already pinned in the code, so this
   is a same-behaviour swap) into `assets/js/vendor/` and pointing
   `dc-runtime.js` at those local paths instead of unpkg. The page no
   longer depends on any third-party CDN to render at all. A small,
   dependency-free fallback (`showBootFailureFallback()`) was also added
   so that if boot ever fails for some other reason, visitors see a
   phone number instead of a blank screen.
2. **The real bug behind the leaked text**: `dc-runtime.js`'s `boot()`
   renders the page correctly on first load (via `parseDcDocument()`,
   which uses `doc.querySelector("x-dc")` against the parsed DOM — safe).
   It then *also* re-fetches the page's own raw HTML a second time
   (`fetch(location.href)`, presumably for the visual editor's
   live-preview/streaming feature) and re-parses that raw text with
   `parseDcText()`, which locates the template boundaries using a plain
   regex (`/<x-dc(?:\s[^>]*)?>/`) with no awareness of HTML comments.
   This file's own previous developer comment happened to contain the
   literal text `<x-dc>` inside an HTML comment (documenting that the
   consent banner sits "outside `<x-dc>`") — the regex matched *that*
   instead of the real opening tag, and sliced everything from
   mid-comment through to the real closing tag into one corrupted
   "template" string, which then got pushed via `runtime.updateHtml()`,
   overwriting the correctly-rendered page a moment after first paint.
   That's why the initial render looked fine and it then went blank with
   leaked comment text on top. **Fixed** by rewording the comment to
   avoid the literal tag-shaped substring, with a warning left in place
   for future editors. This is an upstream bug in `dc-runtime.js` itself
   (in the generated bundle — the real TypeScript source isn't in this
   repo) and should ideally be fixed at the source by making
   `parseDcText()` comment-aware, or by having `boot()` skip that second
   self-fetch/re-render entirely on a plain production deploy — flagging
   this for whoever owns the `dc-runtime` package.

Both fixes were verified headlessly (Playwright/Chromium, no network
access) against all six `?ag=` cases (`core`, `single`, `fullarch`,
`brand`, `finance`, an unrecognised value) with zero console/page errors
and no leaked text in any of them.

## Outstanding — needs Skalr / the clinic, not a developer

These are business confirmations and infrastructure steps that
require accounts, credentials, or facts this build doesn't have
access to:

1. **Confirm the clinic's FCA authorisation status** and lock it into
   the `fcaStatusLine` placeholder in `index.html` (search for
   `TODO(Skalr)`). This is the single most urgent item — the live 0%
   finance ads shouldn't point `?ag=finance` at this page until it's
   filled in.
2. **Swap the CRM endpoint** from the interim Formspree endpoint
   (`https://formspree.io/f/xkoddegp`) to the GoHighLevel
   `ghl-suttonsmiles` inbound webhook, if/when that's ready (search for
   `CRM_ENDPOINT_URL` in `index.html`, and the equivalent `fetch(...)`
   call in `meta-ads.html`).
3. **Lock one review-count figure** — 370+ / 340+ / 320+ currently
   disagree across sources; this build kept the existing 370+/4.9
   figures used throughout `index.html`. Confirm and, if it changes,
   it appears in the hero, trust strip, and price-section CTA.
4. **Confirm GDC No. 103973 belongs to Puneet** (two other numbers,
   273540 and 289701, appear elsewhere for other clinicians).
5. **Supply**: country of qualification, NHS/private status, a real
   privacy-policy URL (currently
   `https://www.suttonsmiles.com/privacy-policy` as a placeholder —
   confirm or replace), clinic email/address/hours accuracy, and any
   real media replacements still needed for before/afters.
6. **GTM container-side work**: finish Consent Mode per-tag gating,
   and set up Meta CAPI server-side (a web GTM container alone can't
   fire it — needs a server-side GTM container or a CRM-side
   integration).
7. **Infrastructure** (needs your GitHub/Netlify/DNS access, not
   something this environment can do on your behalf):
   - Push this repo to a private GitHub repo and add the remote.
   - Connect Netlify to the repo (static, publish `.`, no build
     command — `netlify.toml` is already set up for this).
   - Add the custom domain `lp.suttonsmiles.com` (DNS + Netlify custom
     domain + HTTPS).
   - Point the finance RSAs at `?ag=finance` and the rest at their
     `?ag=` variants once FCA status is confirmed; re-point sitelinks,
     add callouts/snippets (section 08 of the brief — account-side,
     not code).
   - Run the `/skalr-uk` compliance gate before anything goes live.
8. `meta-ads.html` in this repo is a separate Meta-ads variant that
   wasn't touched — it's outside the scope of this Google Ads brief,
   but if it needs the same finance-gating/GDC-footer/consent
   treatment, flag it and it can be brought into line the same way.
9. `vercel.json` is a leftover from an earlier Vercel deploy attempt;
   harmless to keep, safe to delete once Netlify is confirmed as the
   only host.

## Repo status

Git has been initialised locally with one commit. To push:

```
git remote add origin <your-private-github-url>
git branch -M main
git push -u origin main
```
