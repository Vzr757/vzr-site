# Veziroglu Consulting — Website

Static marketing site for **consulting.veziroglu.co.uk**. Plain HTML/CSS, no framework,
no build step. Served from Cloudflare Workers static assets.

## Layout

```
public/                  <- the deployed site; NOTHING else is served
  index.html             <- homepage (#practices #products #work #about #contact)
                            styles are INLINE in a <style> block, not an external file
  404.html               <- served on any unmatched path; inline styles, noindex
  assets/                <- case-study.css, case-study.js (case study pages only)
  images/                <- .webp imagery, favicons
  case-studies/          <- one standalone page per case study
wrangler.jsonc           <- Worker config
package.json / -lock     <- wrangler only; no application deps
```

`public/` is the asset directory. Repo tooling (config, lockfile, this doc) lives at the
root and is deliberately never deployed.

## Deploying

**Push to `main` and it goes live in ~20s.** Cloudflare Workers Builds is connected to
`Vzr757/vzr-site`; it runs `npm ci` then `npx wrangler deploy`. No manual step.

`npx wrangler deploy` still works locally if needed (run it from the repo root).

### The Worker name must stay `vzr-site`

`wrangler.jsonc` → `"name": "vzr-site"` targets the **existing** Worker and updates it in
place. If that name ever drifts, wrangler will happily **create a second Worker** and
deploy there — the live site just silently stops changing while everything reports
success. If a deploy "works" but nothing updates, check this first.

`package-lock.json` is committed because the CI build runs `npm ci`, which fails without
it. There is no build step otherwise.

## Conventions

### Links: no `.html`

Cloudflare serves assets at extensionless paths and **307-redirects** anything ending in
`.html`. Write links as the canonical URL:

```html
<a href="case-studies/document-management">   <!-- yes -->
<a href="case-studies/document-management.html">  <!-- no: costs a 307 -->
<a href="/#work">                             <!-- yes -->
<a href="../index.html#work">                 <!-- no: costs a 307 -->
```

Nothing breaks if you get this wrong, which is exactly why it creeps back in.

### Voice: the firm, not the individual

Copy is written as **"we"**, never "I" — these are engagements by the practice. Matches
the homepage register ("what we deliver", "what we've built").

The one deliberate exception: the **testimonial quote** on the Enterprise Journey
Architecture page is a third party's own words ("I saw him…"). Leave first person intact
there — rewriting it would misrepresent the quote.

### Case studies

Each is a standalone page in `public/case-studies/` sharing `assets/case-study.css`:
hero → key numbers → 01 Context → 02 Challenge → 03 Approach → 04 Outcome → contact.

Sections animate in via `[data-reveal]` + IntersectionObserver. Each page carries a
`<noscript>` fallback that forces them visible — **without it the page is blank if JS
fails.** Keep it when adding pages. `index.html` does not yet have this.

Source of truth is a **Claude Design** project, not these files:
`https://claude.ai/design/p/aadc65fe-f47b-4d01-80d2-9d8116d27c06`
Pages were converted from `.dc.html` (which wrap content in `<x-dc>`/`<helmet>` and a
`DCLogic` script) into plain HTML. Substantial content changes may be better made at
source and re-converted.

## Gotchas

**Keep `not_found_handling: "404-page"`.** Unmatched paths serve `public/404.html` with a
real 404. It was briefly `"single-page-application"`, which made *every* path return 200
with the homepage — dead links rendered silently, missing assets "passed" health checks,
and search engines saw soft 404s. Do not set it back; this site is multi-page, not an SPA.

Clean URLs (`/case-studies/foo`) come from `html_handling`, a separate setting, and are
unaffected by this one.

`404.html` styles itself inline on purpose: an error page that depends on an external
stylesheet breaks in exactly the case where the stylesheet is what went missing.

**Edge caching will lie to you.** After a push, a stale page does *not* mean the deploy
failed. Verify with a cache-buster before debugging:

```bash
curl -s "https://consulting.veziroglu.co.uk/?cb=$RANDOM" | grep ...
curl -s -o /dev/null -D - https://consulting.veziroglu.co.uk/ | grep -i cf-cache-status
```

Or hit `https://vzr-site.emrah-e15.workers.dev/` directly.

**Google Drive.** The working copy lives in a synced Drive folder. Git works fine, but
expect CRLF warnings on commit.
