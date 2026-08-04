# Jekyll Redesign Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rebuild `sakshi-bhalla.github.io` as a slim, self-authored Jekyll theme that reproduces the recovered static design, with content in markdown and YAML and deployment via GitHub Actions.

**Architecture:** One layout, four includes, one stylesheet. Every person- and site-specific string lives in `_config.yml` or `_data/`, never in a template, so the theme layer can later be lifted into a standalone repo unchanged. Publications are structured data rendered by an include; prose, news, and courses are hand-written markdown.

**Tech Stack:** Jekyll 4.4.1, kramdown, `jekyll-sitemap` 1.4.0, `webrick` 1.9.2, `html-proofer` for link/HTML verification, Homebrew Ruby 3.x, GitHub Actions Pages deployment.

**Spec:** `docs/superpowers/specs/2026-08-03-jekyll-redesign-design.md`

## Global Constraints

- Branch is `redesign`. Never commit to `main`. Final deliverable is a PR into `main`.
- **No file under `_layouts/`, `_includes/`, or `assets/` may contain a person- or site-specific string.** Not "Sakshi", "Bhalla", "illinois", "sakshib3", "G-CC9LEWNPER", or any social handle. This is verified in Task 7 and is the single most important rule in this plan.
- Every optional value is guarded: absent config key → the markup is not emitted at all. No empty `<p>`, no bare `mailto:`, no analytics snippet without a tracking id.
- All internal URLs in templates pass through `relative_url`. In CSS, font paths are relative (`../fonts/…`), which is baseurl-safe without Liquid.
- Page prose comes from the recovered static site (`git show static-site-reference:sakshi-site-redesign.zip`), not from `main`. Its wording differs deliberately: "PhD candidate" not "PhD student", no "Prof." honorific, sentence-case publication titles, and "National Institute on Drug Abuse" (the live site's "Drug Addiction" is factually wrong).
- Homebrew Ruby is required. macOS stock Ruby is 2.6; Jekyll 4 needs ≥ 3.0. Every `bundle`/`jekyll` command in this plan assumes `export PATH="/opt/homebrew/opt/ruby/bin:$PATH"` is active in the shell — Homebrew's Ruby is keg-only and is not on `PATH` by default.
- Colors are exact. Light: `--ink #1a1a1a`, `--paper #fffefc`, `--accent #8a2f22`, `--muted #6f6a64`, `--rule #e7e2da`. Dark: `--ink #e9e4dc`, `--paper #14120f`, `--accent #e08f7a`, `--muted #a09890`, `--rule #322e29`. These dark values are already contrast-checked (see Task 3); do not substitute.
- Commit after every task. Conventional-commit prefixes (`feat:`, `chore:`, `docs:`).

## File Structure

| File | Responsibility |
|---|---|
| `Gemfile`, `Gemfile.lock` | Pin the toolchain. Lockfile is committed — CI caches off it. |
| `_config.yml` | Site identity, author identity, social URLs, plugins, excludes. The only place personal strings live besides `_data/` and pages. |
| `_data/navigation.yml` | Nav items: title + url. |
| `_data/social.yml` | Ordered footer links: key + label + optional href prefix. Decouples link order/labels from the template. |
| `_data/publications.yml` | Every paper and work-in-progress. |
| `_layouts/default.html` | The only layout. Assembles head + masthead + content + footer. |
| `_includes/head.html` | `<head>` contents: meta, title, favicons, stylesheet, analytics. |
| `_includes/masthead.html` | Name, role line, nav with self-maintaining `aria-current`. |
| `_includes/footer.html` | Social links generated from `_data/social.yml` × `site.author`. |
| `_includes/publication.html` | Renders one publication entry. |
| `assets/css/style.css` | The entire design. Ported from the static site + `@font-face` + dark mode. |
| `assets/fonts/*.woff2`, `OFL.txt` | Self-hosted type. |
| `index.md` | About prose + News. |
| `research.md` | Lede + two loops over publications data. |
| `teaching.md` | Course list. |
| `cv.md` | One line linking the PDF. Kept because `/cv/` is live and sitemapped. |
| `.github/workflows/pages.yml` | Build and deploy on push to `main`. |

---

### Task 1: Toolchain, Gemfile, repo hygiene, first successful build

**Files:**
- Create: `Gemfile`, `Gemfile.lock`, `_config.yml`, `index.md`
- Modify: `.gitignore`
- Delete: `files/bibtex1.bib`, `files/slides1.pdf`, `files/slides2.pdf`, `files/slides3.pdf`

**Interfaces:**
- Consumes: nothing.
- Produces: a buildable Jekyll site. Later tasks rely on `_config.yml` keys `site.title`, `site.description`, `site.google_analytics`, `site.google_site_verification`, and the `site.author` map with keys `name`, `short_name`, `role`, `email`, `googlescholar`, `github`, `bluesky`, `orcid`, `linkedin`.

- [ ] **Step 1: Install Homebrew Ruby**

```bash
brew install ruby
export PATH="/opt/homebrew/opt/ruby/bin:$PATH"
ruby -v
```

Expected: `ruby 3.x.x`. If it still prints 2.6.10, the `PATH` export did not take — Homebrew's Ruby is keg-only, so `/opt/homebrew/opt/ruby/bin` must come first. Every subsequent step assumes this `PATH`.

- [ ] **Step 2: Write the Gemfile**

Create `Gemfile`:

```ruby
source "https://rubygems.org"

gem "jekyll", "~> 4.4.1"

group :jekyll_plugins do
  gem "jekyll-sitemap", "~> 1.4"
end

# Ruby 3 removed webrick from stdlib; `jekyll serve` needs it.
gem "webrick", "~> 1.9"

group :development do
  gem "html-proofer", "~> 5.0"
end
```

- [ ] **Step 3: Install gems and confirm the lockfile resolves**

```bash
bundle install
bundle exec jekyll -v
```

Expected: `jekyll 4.4.1`. `Gemfile.lock` now exists.

- [ ] **Step 4: Fix `.gitignore` so the lockfile is tracked**

Replace the entire contents of `.gitignore` with:

```gitignore
_site/
.jekyll-cache/
.jekyll-metadata
```

The removed `Gemfile.lock` line is the important one: with `bundler-cache: true`, CI keys its gem cache off the lockfile, and an untracked lockfile means CI resolves fresh every run and can build with different gem versions than local. The `.sass-cache/`, `/vendor/`, `.bundle/`, `node_modules`, `package-lock.json`, and `.vscode/` stanzas are dropped — none apply to this tree.

- [ ] **Step 5: Write `_config.yml`**

```yaml
title: "Sakshi Bhalla"
description: "Sakshi Bhalla is a PhD candidate in Media and Communications at the University of Illinois Urbana-Champaign studying political information environments in India and the United States."
url: "https://sakshi-bhalla.github.io"
baseurl: ""
repository: "sakshi-bhalla/sakshi-bhalla.github.io"
lang: "en"

google_site_verification: "bKREFdenzTY8ptBTygpVLOuE54BlbVu-K8U5014Lm_w"
google_analytics: "G-CC9LEWNPER"

author:
  name: "Sakshi Bhalla"
  short_name: "S. Bhalla"
  role: "PhD Candidate, Media & Communications · University of Illinois Urbana-Champaign"
  email: "sakshib3@illinois.edu"
  googlescholar: "https://scholar.google.com/citations?user=QewpAPIAAAAJ&hl=en"
  github: "https://github.com/sakshi-bhalla"
  bluesky: "https://bsky.app/profile/sakshi-bhalla.bsky.social"
  orcid: "https://orcid.org/0000-0002-6803-7657"
  linkedin: "https://www.linkedin.com/in/sb11"

markdown: kramdown
kramdown:
  input: GFM
  smart_quotes: lsquo,rsquo,ldquo,rdquo

plugins:
  - jekyll-sitemap

exclude:
  - Gemfile
  - Gemfile.lock
  - docs
  - README.md
  - vendor
```

Note there is no `twitter` key. That absence is what removes the X link from the footer — the template stays generic. `docs` must be excluded or this plan and its spec get published at `sakshi-bhalla.github.io/docs/…`.

- [ ] **Step 6: Write a minimal `index.md` so the build has a page**

```markdown
---
permalink: /
---

Placeholder.
```

- [ ] **Step 7: Delete the academicpages placeholder files**

```bash
git rm files/bibtex1.bib files/slides1.pdf files/slides2.pdf files/slides3.pdf
```

These are template dummies — `bibtex1.bib` cites "Alice, Bob and Charlie" in the *Journal of Examples*, and each slide deck is a 2-page PDF whose only text is "Slides 1/2/3". Nothing links them, but the live `sitemap.xml` advertises all four. `cv.pdf` and `paper1-3.pdf` are real; leave them.

- [ ] **Step 8: Build and verify**

```bash
bundle exec jekyll build
test -f _site/index.html && echo "index OK"
test -f _site/sitemap.xml && echo "sitemap OK"
grep -c "slides1.pdf\|bibtex1" _site/sitemap.xml
ls _site/docs 2>/dev/null && echo "FAIL: docs was published" || echo "docs excluded OK"
```

Expected: `index OK`, `sitemap OK`, grep count `0`, `docs excluded OK`. The build must print no warnings.

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "chore: add Jekyll 4 toolchain and drop academicpages placeholders"
```

---

### Task 2: Layout, includes, and navigation data

**Files:**
- Create: `_data/navigation.yml`, `_data/social.yml`, `_includes/head.html`, `_includes/masthead.html`, `_includes/footer.html`, `_layouts/default.html`
- Modify: `index.md`, `images/manifest.json`

**Interfaces:**
- Consumes: `site.title`, `site.description`, `site.author.*`, `site.google_analytics`, `site.google_site_verification` from Task 1.
- Produces: the `default` layout, which every page from Tasks 4–5 declares as `layout: default`. Pages set `title` (used for `<title>` and nothing else — headings are written in the page body) and optionally `description` and `permalink`. The DOM contract the stylesheet in Task 3 targets: `div.page > header.masthead` (containing `h1`, `p.role`, `nav`), `main`, `footer`.

- [ ] **Step 1: Write `_data/navigation.yml`**

```yaml
- title: About
  url: /
- title: Research
  url: /research/
- title: Teaching
  url: /teaching/
- title: CV
  url: /files/cv.pdf
```

The CV entry points at the PDF, matching the static design. `/cv/` exists as a page (Task 4) but is intentionally not in the nav — it is kept alive for old links and search results.

- [ ] **Step 2: Write `_data/social.yml`**

```yaml
- key: email
  label: Email
  prefix: "mailto:"
- key: googlescholar
  label: Scholar
- key: github
  label: GitHub
- key: bluesky
  label: Bluesky
- key: orcid
  label: ORCID
- key: linkedin
  label: LinkedIn
- key: twitter
  label: X
```

Order and labels live here, URLs live in `site.author`. A row whose `site.author[key]` is unset renders nothing — which is exactly how the X link stays off the page while the theme keeps supporting it.

- [ ] **Step 3: Write `_includes/head.html`**

```html
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>{% if page.title %}{{ page.title | escape }} · {% endif %}{{ site.title | escape }}</title>
{% assign page_description = page.description | default: site.description %}
{% if page_description %}<meta name="description" content="{{ page_description | escape }}">{% endif %}
{% if site.google_site_verification %}<meta name="google-site-verification" content="{{ site.google_site_verification }}">{% endif %}
<link rel="canonical" href="{{ page.url | absolute_url }}">
<meta name="theme-color" content="#fffefc" media="(prefers-color-scheme: light)">
<meta name="theme-color" content="#14120f" media="(prefers-color-scheme: dark)">
<link rel="icon" href="{{ '/images/favicon.ico' | relative_url }}" sizes="any">
<link rel="icon" href="{{ '/images/favicon.svg' | relative_url }}" type="image/svg+xml">
<link rel="icon" href="{{ '/images/favicon-32x32.png' | relative_url }}" type="image/png" sizes="32x32">
<link rel="icon" href="{{ '/images/favicon-192x192.png' | relative_url }}" type="image/png" sizes="192x192">
<link rel="apple-touch-icon" href="{{ '/images/apple-touch-icon-180x180.png' | relative_url }}" sizes="180x180">
<link rel="manifest" href="{{ '/images/manifest.json' | relative_url }}">
<link rel="stylesheet" href="{{ '/assets/css/style.css' | relative_url }}">
{% if site.google_analytics %}
<script async src="https://www.googletagmanager.com/gtag/js?id={{ site.google_analytics }}"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', '{{ site.google_analytics }}');
</script>
{% endif %}
```

- [ ] **Step 4: Write `_includes/masthead.html`**

```html
<header class="masthead">
  <h1><a href="{{ '/' | relative_url }}">{{ site.title | escape }}</a></h1>
  {% if site.author.role %}<p class="role">{{ site.author.role | escape }}</p>{% endif %}
  <nav>
    {% for item in site.data.navigation %}
    <a href="{{ item.url | relative_url }}"{% if page.url == item.url %} aria-current="page"{% endif %}>{{ item.title | escape }}</a>
    {% endfor %}
  </nav>
</header>
```

`page.url == item.url` is what makes the active-page highlight maintain itself. It never matches the CV row, because `/files/cv.pdf` is not a page — correct, since a PDF link should not be marked as the current page.

- [ ] **Step 5: Write `_includes/footer.html`**

```html
<footer>
  {% for item in site.data.social %}
  {% assign social_url = site.author[item.key] %}
  {% if social_url %}<a href="{{ item.prefix }}{{ social_url }}">{{ item.label | escape }}</a>{% endif %}
  {% endfor %}
</footer>
```

- [ ] **Step 6: Write `_layouts/default.html`**

```html
<!DOCTYPE html>
<html lang="{{ site.lang | default: 'en' }}">
<head>
{% include head.html %}
</head>
<body>
<div class="page">
{% include masthead.html %}
<main>
{{ content }}
</main>
{% include footer.html %}
</div>
</body>
</html>
```

- [ ] **Step 7: Point `index.md` at the layout**

Replace `index.md` with:

```markdown
---
layout: default
permalink: /
---

Placeholder.
```

- [ ] **Step 8: Fix the favicon manifest name**

In `images/manifest.json`, change `"name": "OOjs UI icon academic-progressive"` to `"name": "Sakshi Bhalla"`. The current value is a leftover from the icon's Wikimedia source and is what a browser shows if the site is installed to a home screen.

- [ ] **Step 9: Build and verify the chrome renders**

```bash
bundle exec jekyll build
grep -c 'aria-current="page"' _site/index.html          # expect 1
grep -o 'href="/files/cv.pdf"' _site/index.html         # expect a match
grep -c 'X</a>' _site/index.html                        # expect 0 — twitter key is unset
grep -c 'gtag' _site/index.html                         # expect >0
grep -c 'OOjs' _site/images/manifest.json               # expect 0
```

Expected exactly: `1`, a match, `0`, a nonzero count, `0`.

- [ ] **Step 10: Verify the no-hardcoding rule holds so far**

```bash
grep -rniE 'sakshi|bhalla|illinois|sakshib3|G-CC9LEWNPER|sabyasakshi|orcid\.org|bsky' _layouts/ _includes/ && echo "FAIL" || echo "clean"
```

Expected: `clean`. If this fails, the offending string belongs in `_config.yml` or `_data/`, not the template.

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "feat: add layout, includes, and navigation data"
```

---

### Task 3: Self-hosted fonts, stylesheet, and dark mode

**Files:**
- Create: `assets/fonts/` (5 × `.woff2` + `OFL-SourceSerif4.txt` + `OFL-IBMPlexSans.txt`), `assets/css/style.css`

**Interfaces:**
- Consumes: the DOM contract from Task 2 — `div.page`, `header.masthead > h1 / p.role / nav`, `main`, `footer`.
- Produces: the class names Tasks 4–5 write into markup: `.entry` (with a child `.date`), `.pub` (with children `.title`, `.byline`, `.venue`, `.links`, and a `<details>`/`<summary>` pair).

- [ ] **Step 1: Download the fonts**

```bash
mkdir -p assets/fonts
SS=https://cdn.jsdelivr.net/npm/@fontsource/source-serif-4@5.2.6
PX=https://cdn.jsdelivr.net/npm/@fontsource/ibm-plex-sans@5.2.6
curl -sS -o assets/fonts/source-serif-4-latin-400-normal.woff2 "$SS/files/source-serif-4-latin-400-normal.woff2"
curl -sS -o assets/fonts/source-serif-4-latin-600-normal.woff2 "$SS/files/source-serif-4-latin-600-normal.woff2"
curl -sS -o assets/fonts/source-serif-4-latin-400-italic.woff2 "$SS/files/source-serif-4-latin-400-italic.woff2"
curl -sS -o assets/fonts/ibm-plex-sans-latin-400-normal.woff2  "$PX/files/ibm-plex-sans-latin-400-normal.woff2"
curl -sS -o assets/fonts/ibm-plex-sans-latin-500-normal.woff2  "$PX/files/ibm-plex-sans-latin-500-normal.woff2"
curl -sS -o assets/fonts/OFL-SourceSerif4.txt "$SS/LICENSE"
curl -sS -o assets/fonts/OFL-IBMPlexSans.txt  "$PX/LICENSE"
ls -la assets/fonts/
```

Expected: five `.woff2` files, roughly 20–24 KB each (~107 KB total), and two license files. Both families are SIL Open Font License; shipping the license text is a condition of redistributing them.

- [ ] **Step 2: Verify each font file is really a woff2**

```bash
for f in assets/fonts/*.woff2; do printf "%s " "$f"; head -c 4 "$f" | xxd -p; done
```

Expected: each line ends in `774f4632` (`wOF2`). Anything else means curl saved an error page.

- [ ] **Step 3: Write `assets/css/style.css`**

This is the static site's stylesheet, ported. Font paths are `../fonts/…` — relative to `/assets/css/`, so they survive any `baseurl` without needing Liquid in the CSS.

```css
/* Type: Source Serif 4 (text) + IBM Plex Sans (metadata) */

@font-face {
  font-family: "Source Serif 4";
  font-style: normal;
  font-weight: 400;
  font-display: swap;
  src: url("../fonts/source-serif-4-latin-400-normal.woff2") format("woff2");
}
@font-face {
  font-family: "Source Serif 4";
  font-style: normal;
  font-weight: 600;
  font-display: swap;
  src: url("../fonts/source-serif-4-latin-600-normal.woff2") format("woff2");
}
@font-face {
  font-family: "Source Serif 4";
  font-style: italic;
  font-weight: 400;
  font-display: swap;
  src: url("../fonts/source-serif-4-latin-400-italic.woff2") format("woff2");
}
@font-face {
  font-family: "IBM Plex Sans";
  font-style: normal;
  font-weight: 400;
  font-display: swap;
  src: url("../fonts/ibm-plex-sans-latin-400-normal.woff2") format("woff2");
}
@font-face {
  font-family: "IBM Plex Sans";
  font-style: normal;
  font-weight: 500;
  font-display: swap;
  src: url("../fonts/ibm-plex-sans-latin-500-normal.woff2") format("woff2");
}

:root {
  --ink: #1a1a1a;
  --paper: #fffefc;
  --accent: #8a2f22;
  --muted: #6f6a64;
  --rule: #e7e2da;
  --underline: #c9a49e;
  --abstract: #3a3733;
  --select: #f3dcd6;
  --serif: "Source Serif 4", Charter, Georgia, serif;
  --sans: "IBM Plex Sans", system-ui, sans-serif;
}

@media (prefers-color-scheme: dark) {
  :root {
    --ink: #e9e4dc;
    --paper: #14120f;
    --accent: #e08f7a;
    --muted: #a09890;
    --rule: #322e29;
    --underline: #8a5c50;
    --abstract: #c9c3ba;
    --select: #4a2f28;
  }
}

* { box-sizing: border-box; }

html { font-size: 18px; }

body {
  margin: 0;
  background: var(--paper);
  color: var(--ink);
  font-family: var(--serif);
  line-height: 1.6;
  -webkit-font-smoothing: antialiased;
  text-rendering: optimizeLegibility;
}

.page {
  max-width: 42rem;
  margin: 0 auto;
  padding: 3.5rem 1.25rem 5rem;
}

.masthead { margin-bottom: 3rem; }

.masthead h1 {
  font-size: 1.55rem;
  font-weight: 600;
  letter-spacing: 0.01em;
  margin: 0 0 0.2rem;
}

.masthead h1 a { color: var(--ink); text-decoration: none; }

.masthead .role {
  font-size: 0.98rem;
  color: var(--muted);
  margin: 0 0 1.1rem;
}

.masthead nav {
  font-family: var(--sans);
  font-size: 0.72rem;
  letter-spacing: 0.14em;
  text-transform: uppercase;
  border-bottom: 1px solid var(--rule);
  padding-bottom: 1.1rem;
}

.masthead nav a {
  color: var(--muted);
  text-decoration: none;
  margin-right: 1.4rem;
}

.masthead nav a:hover { color: var(--accent); }
.masthead nav a[aria-current="page"] { color: var(--ink); }

h2 {
  font-size: 1.15rem;
  font-weight: 600;
  margin: 2.6rem 0 0.9rem;
}

h2:first-of-type { margin-top: 0; }

p { margin: 0 0 1.05rem; }

a {
  color: var(--accent);
  text-decoration: underline;
  text-decoration-thickness: 1px;
  text-underline-offset: 2.5px;
  text-decoration-color: var(--underline);
}

a:hover { text-decoration-color: var(--accent); }

/* Dated entries (news, publications): date hangs in the left margin */
.entry {
  position: relative;
  margin: 0 0 1.35rem;
}

.entry .date {
  font-family: var(--sans);
  font-size: 0.7rem;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: var(--muted);
  display: block;
  margin-bottom: 0.15rem;
}

@media (min-width: 62rem) {
  .entry .date {
    position: absolute;
    left: -9.5rem;
    top: 0.28rem;
    width: 8.25rem;
    text-align: right;
    margin: 0;
  }
}

.pub { margin-bottom: 1.6rem; }

.pub .title {
  font-weight: 600;
  margin-bottom: 0.1rem;
}

.pub .byline {
  font-size: 0.93rem;
  margin-bottom: 0.15rem;
}

.pub .venue { font-style: italic; }

.pub .links {
  font-family: var(--sans);
  font-size: 0.72rem;
  letter-spacing: 0.05em;
  text-transform: uppercase;
}

.pub .links a { text-decoration: none; color: var(--accent); }
.pub .links a:hover { text-decoration: underline; }

.pub details { margin-top: 0.25rem; }

.pub summary {
  font-family: var(--sans);
  font-size: 0.72rem;
  letter-spacing: 0.05em;
  text-transform: uppercase;
  color: var(--muted);
  cursor: pointer;
  list-style: none;
}

.pub summary::-webkit-details-marker { display: none; }
.pub summary::after { content: " +"; }
.pub details[open] summary::after { content: " \2212"; }

.pub details p {
  font-size: 0.93rem;
  color: var(--abstract);
  margin-top: 0.35rem;
}

ul { padding-left: 1.1rem; margin: 0 0 1.05rem; }
li { margin-bottom: 0.35rem; }

footer {
  margin-top: 4rem;
  padding-top: 1.1rem;
  border-top: 1px solid var(--rule);
  font-family: var(--sans);
  font-size: 0.72rem;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  color: var(--muted);
}

footer a { color: var(--muted); text-decoration: none; margin-right: 1.2rem; }
footer a:hover { color: var(--accent); }

::selection { background: var(--select); }

:focus-visible { outline: 2px solid var(--accent); outline-offset: 2px; }

@media (prefers-reduced-motion: reduce) {
  * { animation: none !important; transition: none !important; }
}
```

Four deliberate departures from the original file, all required by the spec: the `@font-face` block replaces the Google Fonts `<link>`; `--underline`, `--abstract`, and `--select` are promoted from hardcoded hex values to custom properties so the dark block can override them without duplicating a single rule; the dark block itself is new; and `img.panel` is deleted — it was styled but used by no page.

- [ ] **Step 4: Confirm the dark palette meets WCAG AA**

```bash
python3 - <<'PY'
def lin(c):
    c /= 255
    return c / 12.92 if c <= 0.04045 else ((c + 0.055) / 1.055) ** 2.4
def L(h):
    h = h.lstrip('#'); r, g, b = (int(h[i:i+2], 16) for i in (0, 2, 4))
    return 0.2126 * lin(r) + 0.7152 * lin(g) + 0.0722 * lin(b)
def cr(a, b):
    la, lb = L(a), L(b); hi, lo = max(la, lb), min(la, lb)
    return (hi + 0.05) / (lo + 0.05)
for paper, pairs in [
    ('#fffefc', [('ink', '#1a1a1a'), ('muted', '#6f6a64'), ('accent', '#8a2f22'), ('abstract', '#3a3733')]),
    ('#14120f', [('ink', '#e9e4dc'), ('muted', '#a09890'), ('accent', '#e08f7a'), ('abstract', '#c9c3ba')]),
]:
    for name, c in pairs:
        r = cr(c, paper)
        print(f"{paper} {name:9s} {c} {r:5.2f} {'PASS' if r >= 4.5 else 'FAIL'}")
PY
```

Expected: every line `PASS`. Reference values — light: ink 17.27, muted 5.31, accent 8.30, abstract 11.74; dark: ink 14.78, muted 6.58, accent 7.46, abstract 10.68.

`--rule` (1.28 light / 1.39 dark) and `--underline` (2.24 / 3.32) are deliberately excluded from this gate. Both are decorative hairlines, not text and not UI controls, so WCAG 1.4.3 and 1.4.11 do not apply; the link text itself carries the ≥ 7:1 accent contrast. The dark values are tuned to sit at the same visual weight as the light originals.

- [ ] **Step 5: Build and verify the CSS ships with working font references**

```bash
bundle exec jekyll build
test -f _site/assets/css/style.css && echo "css OK"
ls _site/assets/fonts/*.woff2 | wc -l          # expect 5
grep -c "fonts.googleapis" _site/assets/css/style.css _site/index.html   # expect 0 for both
```

- [ ] **Step 6: Look at it in both color schemes**

```bash
bundle exec jekyll serve --port 4000 --detach
for scheme in light dark; do
  playwright screenshot --channel=chrome --color-scheme=$scheme \
    --viewport-size=1280,900 http://localhost:4000/ "/tmp/task3-$scheme.png"
done
md5 /tmp/task3-light.png /tmp/task3-dark.png
```

Use Playwright, not `chrome --headless --force-prefers-color-scheme`. That Chrome
flag is silently a no-op — it produces two byte-identical light-mode PNGs and
will make a broken dark mode look verified. The `md5` line is the guard: if the
two hashes match, the emulation did not take. `--channel=chrome` reuses the
installed Google Chrome, so no browser download is needed.

Open both PNGs and actually look at them. A blank or unstyled frame is a failure, not a pass. Expect: centered column, serif body, uppercase letterspaced nav — and in dark, warm near-black paper with legible salmon links.

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: add self-hosted fonts, stylesheet, and dark mode"
```

---

### Task 4: Content pages — about, teaching, CV

**Files:**
- Modify: `index.md`
- Create: `teaching.md`, `cv.md`

**Interfaces:**
- Consumes: `layout: default` from Task 2; `.entry` / `.date` from Task 3.
- Produces: `/`, `/teaching/`, `/cv/`. Task 5 adds `/research/`, the only nav target still missing after this task.

- [ ] **Step 1: Write `index.md`**

Page body carries no `<h1>` — the masthead already supplies one, and a second would be a document-outline error.

```markdown
---
layout: default
permalink: /
---

I am a PhD candidate in Media and Communications at the [University of Illinois Urbana-Champaign](https://media.illinois.edu/sakshi-bhalla/), advised by [Scott Althaus](https://clinecenter.illinois.edu/people/salthaus). I hold the Illinois Distinguished Fellowship, the university's most prestigious award for incoming graduate students.

I study political information: its political economy and its spillovers onto political behavior in democratic contexts, particularly India and the United States. My work examines how the structure and dynamics of the information environment shape who sees what, when, and how, and whether the evolving relationship between media supply and audience demand in high-choice environments serves democratic informational needs.

Methodologically, I rely on econometric methods, including spatial and network analysis and causal inference, alongside qualitative work such as semi-structured interviews and discourse analysis. I earned an MS in Statistics alongside my PhD (completed Summer 2025). Before Illinois, I completed an MA in Linguistics at Jawaharlal Nehru University and a BA (Honors) in Journalism at Lady Shri Ram College, both in New Delhi.

This summer I am at Amazon HQ2 in Washington, DC as a Research Scientist Intern, bringing my work in econometrics, computational social science, and public opinion dynamics to the role. If our paths cross, I am always happy to grab a coffee and discuss shared interests or potential collaborations: sakshib3 [at] illinois [dot] edu.

## News

<div class="entry">
  <span class="date">June 2025</span>
  <p>I will be at the International Communication Association annual meeting in Denver, including the Political Communication Preconference. Let's meet!</p>
</div>

<div class="entry">
  <span class="date">May 2025</span>
  <p>Awarded the Graduate Student Fellowship from the Political Networks methodological society.</p>
</div>

<div class="entry">
  <span class="date">April 2025</span>
  <p>Our study <a href="https://academic.oup.com/joc/advance-article/doi/10.1093/joc/jqaf018/8129746">"Partisan news users in the United States and India on either side seldom use fact checkers"</a> is out at the <em>Journal of Communication</em>. No institutional access? Write to me.</p>
</div>

<div class="entry">
  <span class="date">March 2025</span>
  <p>Received the Methodology Center Summer Institute Scholarship (Purdue University / National Institute on Drug Abuse). Looking forward to West Lafayette in July.</p>
</div>

<div class="entry">
  <span class="date">October 2024</span>
  <p>My first-authored study on why misinformation persists despite corrections, <a href="https://www.tandfonline.com/doi/full/10.1080/1369118X.2024.2406819">"When news is entertainment,"</a> is out at <em>Information, Communication &amp; Society</em>.</p>
</div>

<div class="entry">
  <span class="date">August 2024</span>
  <p>My <a href="https://www.cislm.org/qa-with-local-news-circulation-researcher-sakshi-bhalla/">interview with CISLM</a> on my study of local print circulation is live.</p>
</div>

<div class="entry">
  <span class="date">June 2024</span>
  <p>Presented my working paper on digital and legacy media structures in India, and how they shape audience behavior on social platforms, at the Social Media and Society in India Conference at the University of Michigan. <a href="https://drive.google.com/file/d/1CQmoQ5ym5nRqDBszTJK8P0oVnWvMsaN0/view">Draft here.</a></p>
</div>
```

- [ ] **Step 2: Write `teaching.md`**

```markdown
---
layout: default
title: Teaching
permalink: /teaching/
description: "Courses taught by Sakshi Bhalla at the University of Illinois Urbana-Champaign."
---

## Teaching

- Foundations of Data Curation (CS/IS 598) · Lead Teaching Assistant
- Data Management, Curation & Reproducibility (IS 477) · Teaching Assistant
- Audience Analysis (ADV 483) · Instructor of Record
- Intro to Advertising (ADV 150) · Teaching Assistant
- Intro to Popular TV and Movies (MACS 100) · Teaching Assistant
```

- [ ] **Step 3: Write `cv.md`**

```markdown
---
layout: default
title: CV
permalink: /cv/
description: "Curriculum vitae for Sakshi Bhalla."
---

## CV

Download the full CV as a PDF: [cv.pdf](/files/cv.pdf).
```

- [ ] **Step 4: Build and verify all three routes and their content**

```bash
bundle exec jekyll build
for p in index teaching/index cv/index; do test -f "_site/$p.html" && echo "$p OK" || echo "$p MISSING"; done
grep -c 'class="entry"' _site/index.html        # expect 7
grep -c '<h1' _site/index.html                  # expect 1 — masthead only
grep -c '<li>' _site/teaching/index.html        # expect 5
grep -o '<title>[^<]*' _site/teaching/index.html   # expect "Teaching · Sakshi Bhalla"
grep -o '<title>[^<]*' _site/index.html            # expect "Sakshi Bhalla" alone
```

- [ ] **Step 5: Check nothing was dropped against the live site**

```bash
curl -sS https://sakshi-bhalla.github.io/about/ -o /tmp/live-about.html
python3 - <<'PY'
import re, html
def words(p):
    d = open(p, encoding='utf-8').read()
    d = re.sub(r'(?is)<(script|style|svg|head|nav|footer)\b.*?</\1>', ' ', d)
    d = re.sub(r'<[^>]+>', ' ', d)
    return set(re.findall(r"[A-Za-z][A-Za-z'&-]+", html.unescape(d).lower()))
live, new = words('/tmp/live-about.html'), words('_site/index.html')
print("only on live:", sorted(live - new))
PY
```

Every word reported as "only on live" must be explainable by a deliberate rewrite from the spec — `student` (now "candidate"), `prof` (honorific dropped), `addiction` (corrected to "abuse"), `substantively`/`advances`/`align` (paragraph 2–3 rewrite), `updates` (heading is now "News"), sidebar strings (`urbana`, `follow`), and academicpages chrome (`powered`, `jekyll`, `minimal`, `mistakes`, `sitemap`, `feed`). Anything else is an accidental loss — go back and add it.

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: add about, teaching, and CV pages"
```

---

### Task 5: Publications data, include, and the research page

**Files:**
- Create: `_data/publications.yml`, `_includes/publication.html`, `research.md`

**Interfaces:**
- Consumes: `site.author.short_name` (Task 1) for author bolding; `.pub` / `.entry` styling (Task 3).
- Produces: `/research/`, completing the nav.

- [ ] **Step 1: Write `_data/publications.yml`**

```yaml
- title: "Partisan news users in the United States and India on either side seldom use fact checkers"
  authors: ["R. Ray", "S. Bhalla", "H. Taneja"]
  year: 2025
  venue: "Journal of Communication"
  status: published
  pdf: /files/paper1.pdf
  doi: "https://academic.oup.com/joc/advance-article/doi/10.1093/joc/jqaf018/8129746"
  abstract: >
    Fact checkers have low reach, and their limited efficacy is often attributed to
    perceived partisanship. Yet little research exists investigating the reach of or
    engagement with fact checkers among their intended audiences. We argue that given
    their small audience size, fact checkers' usage is likely driven by heavy media
    users regardless of partisan leanings. Examining over 7 million Twitter (X) news
    users across India and the United States, we find exposure to and engagement with
    fact checkers remains largely restricted to heavier users, with little evidence
    that interventions penetrate selectively partisan news audiences.

- title: "When news is entertainment: Explaining the persistence of misinformation through the information environment"
  authors: ["S. Bhalla", "R. Ray", "H. Taneja"]
  year: 2024
  venue: "Information, Communication & Society"
  status: published
  pdf: /files/paper2.pdf
  doi: "https://www.tandfonline.com/doi/full/10.1080/1369118X.2024.2406819"
  abstract: >
    We propose the "news as entertainment" framework to explain how commercial dynamics
    shape news consumption and why misinformation persists despite corrections. Using
    India's information environment as a strategic case, we show how competing social
    and economic interests in a high-choice, polarized context influence exposure and
    receptivity to corrections.

- title: "Climate strikes in millennial India: Social capital and \"on-ground\" networks in digital-first movements"
  authors: ["A. Khan", "S. Natarajan", "S. Bhalla"]
  year: 2021
  venue: "Communication, Culture & Critique"
  status: published
  pdf: /files/paper3.pdf
  doi: "https://academic.oup.com/ccc/article-abstract/14/3/518/6307130"
  abstract: >
    Using the September 2019 climate strikes in Delhi and Bengaluru, we examine how
    Twitter activity intertwined with local social-capital networks in a digital-first
    movement. We show that the role of Twitter can only be understood alongside
    on-the-ground organizing.

- title: "Classroom contexts: Teachers talk teaching media literacy"
  authors: ["S. Bhalla", "M. Nelson", "M. Spikes"]
  venue: "Journal of Media Literacy Education"
  status: forthcoming
  abstract: >
    Interviews with 20 educators reveal divides in media worlds, school resources, and
    political context that shape how media literacy is taught and experienced,
    highlighting the need for bottom-up approaches to improve outcomes.

- title: "Following the news: Polarization and the networked structure of attention"
  status: wip

- title: "News(paper) flows: A spatial examination of local newspaper circulation"
  status: wip
  note: "Earlier version presented at the Local News Researchers Workshop, March 2024, Durham, NC."
```

The forthcoming entry has no `pdf` and no `doi` — the live site links a bare `https://www.jmle.org`, which is the journal's homepage, not the article. A homepage link labelled DOI is worse than no link, so it is dropped.

- [ ] **Step 2: Write `_includes/publication.html`**

Called as `{% include publication.html pub=item %}`.

```html
<div class="pub entry">
  {% if include.pub.status == 'published' and include.pub.year %}
    <span class="date">{{ include.pub.year }}</span>
  {% elsif include.pub.status == 'forthcoming' %}
    <span class="date">Forthcoming</span>
  {% endif %}
  <p class="title">{{ include.pub.title | escape }}</p>
  {% if include.pub.authors %}
    <p class="byline">
      {%- for author in include.pub.authors -%}
        {%- if author == site.author.short_name -%}<strong>{{ author | escape }}</strong>{%- else -%}{{ author | escape }}{%- endif -%}
        {%- unless forloop.last -%}{%- if forloop.length > 2 -%},{%- endif %} {% if forloop.rindex == 2 -%}and {% endif -%}{%- endunless -%}
      {%- endfor -%}
      {%- if include.pub.venue %} · <span class="venue">{{ include.pub.venue | escape }}</span>{% endif -%}
    </p>
  {% elsif include.pub.note %}
    <p class="byline">{{ include.pub.note | escape }}</p>
  {% endif %}
  {% if include.pub.pdf or include.pub.doi %}
    <p class="links">
      {% if include.pub.pdf %}<a href="{{ include.pub.pdf | relative_url }}">PDF</a>{% endif %}
      {% if include.pub.pdf and include.pub.doi %}&nbsp;{% endif %}
      {% if include.pub.doi %}<a href="{{ include.pub.doi }}">DOI</a>{% endif %}
    </p>
  {% endif %}
  {% if include.pub.abstract %}
    <details>
      <summary>Abstract</summary>
      <p>{{ include.pub.abstract | strip | escape }}</p>
    </details>
  {% endif %}
</div>
```

The author-separator logic produces `A, B, and C` for three or more and `A and B` for two — `forloop.length > 2` gates the comma, `forloop.rindex == 2` inserts the `and` before the final name.

- [ ] **Step 3: Write `research.md`**

```markdown
---
layout: default
title: Research
permalink: /research/
description: "Publications and working papers by Sakshi Bhalla."
---

My current and forthcoming work focuses on structural change in information environments, media and information infrastructures and their consequences for political behavior, and platform-mediated news exposure and polarization.

## Publications

{% for item in site.data.publications %}{% unless item.status == 'wip' %}{% include publication.html pub=item %}{% endunless %}{% endfor %}

## Works in progress

{% for item in site.data.publications %}{% if item.status == 'wip' %}{% include publication.html pub=item %}{% endif %}{% endfor %}
```

- [ ] **Step 4: Build and verify the rendering**

```bash
bundle exec jekyll build
grep -c 'class="pub entry"' _site/research/index.html    # expect 6
grep -c '<details>' _site/research/index.html            # expect 4 — wip entries have no abstract
grep -c '<strong>S. Bhalla</strong>' _site/research/index.html   # expect 4
grep -c 'Forthcoming' _site/research/index.html          # expect 1
grep -o 'R. Ray, <strong>S. Bhalla</strong>, and H. Taneja' _site/research/index.html
grep -c 'jmle.org' _site/research/index.html             # expect 0
```

The `grep -o` must print the byline exactly as shown — that string is the proof the separator logic is right.

- [ ] **Step 5: Verify graceful degradation with no data**

This is the theme-extractability guarantee: a reuser without publications must still get a working page.

```bash
cp _data/publications.yml /tmp/pubs-backup.yml
echo "[]" > _data/publications.yml
bundle exec jekyll build 2>&1 | tee /tmp/emptybuild.log
grep -ci "error\|warn" /tmp/emptybuild.log     # expect 0
test -f _site/research/index.html && echo "research still builds"
grep -c 'class="pub' _site/research/index.html  # expect 0
cp /tmp/pubs-backup.yml _data/publications.yml
bundle exec jekyll build
```

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: add publications data and the research page"
```

---

### Task 6: GitHub Actions deployment workflow

**Files:**
- Create: `.github/workflows/pages.yml`

**Interfaces:**
- Consumes: the `Gemfile.lock` committed in Task 1 (`bundler-cache: true` keys its cache off it).
- Produces: nothing other tasks depend on.

- [ ] **Step 1: Write `.github/workflows/pages.yml`**

Action versions below are the current latest majors, verified against each action's releases on 2026-08-03 — not carried over from an older template.

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: "3.4"
          bundler-cache: true
      - uses: actions/configure-pages@v6
      - name: Build site
        run: bundle exec jekyll build --trace
        env:
          JEKYLL_ENV: production
      - name: Check links and markup
        run: bundle exec htmlproofer ./_site --disable-external --ignore-urls "/^\/files\//"
      - uses: actions/upload-pages-artifact@v5

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v5
```

`--disable-external` keeps CI from failing because a journal's site is briefly down; internal links are what a build can actually break. The `/files/` ignore covers the PDFs, which html-proofer cannot parse.

- [ ] **Step 2: Lint the workflow**

```bash
brew install actionlint
actionlint .github/workflows/pages.yml && echo "workflow lints clean"
```

Expected: no output from `actionlint`, then the success message. If `brew install` is unavailable, `docker run --rm -v "$PWD":/repo -w /repo rhysd/actionlint:latest` is equivalent.

- [ ] **Step 3: Run the same html-proofer check locally that CI will run**

```bash
bundle exec jekyll build
bundle exec htmlproofer ./_site --disable-external --ignore-urls "/^\/files\//"
```

Expected: `HTML-Proofer finished successfully`. Any broken internal link must be fixed now — this is the check that would otherwise fail the deploy.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "ci: deploy to GitHub Pages via Actions"
```

---

### Task 7: Full verification sweep and pull request

**Files:** none created; this task proves the spec's nine verification criteria and opens the PR.

**Interfaces:**
- Consumes: everything.
- Produces: a PR into `main`.

- [ ] **Step 1: Clean build, no warnings**

```bash
rm -rf _site .jekyll-cache
bundle exec jekyll build 2>&1 | tee /tmp/build.log
grep -ci "warn\|error" /tmp/build.log      # expect 0
```

- [ ] **Step 2: Every route serves**

```bash
bundle exec jekyll serve --port 4000 --detach
sleep 3
for u in / /research/ /teaching/ /cv/ /files/cv.pdf /assets/css/style.css \
         /assets/fonts/source-serif-4-latin-400-normal.woff2 \
         /assets/fonts/source-serif-4-latin-600-normal.woff2 \
         /assets/fonts/source-serif-4-latin-400-italic.woff2 \
         /assets/fonts/ibm-plex-sans-latin-400-normal.woff2 \
         /assets/fonts/ibm-plex-sans-latin-500-normal.woff2 \
         /sitemap.xml; do
  printf "%s %s\n" "$(curl -sS -o /dev/null -w '%{http_code}' "http://localhost:4000$u")" "$u"
done
```

Expected: `200` on every line.

- [ ] **Step 3: Sitemap advertises only real things**

```bash
curl -sS http://localhost:4000/sitemap.xml | grep -o '<loc>[^<]*</loc>'
```

Expected: `/`, `/cv/`, `/research/`, `/teaching/`, and the four real PDFs. No `slides`, no `bibtex`, no `/blog/`, no `/docs/`.

- [ ] **Step 4: No third-party font requests survive**

```bash
grep -r "fonts.googleapis\|fonts.gstatic" _site/ && echo "FAIL" || echo "no external font requests"
```

- [ ] **Step 5: Theme-extractability grep — the rule from Global Constraints**

```bash
grep -rniE 'sakshi|bhalla|illinois|sakshib3|G-CC9LEWNPER|sabyasakshi|scholar\.google|orcid\.org|bsky|linkedin' \
  _layouts/ _includes/ assets/css/ && echo "FAIL: personal content in the theme layer" || echo "theme layer clean"
```

Expected: `theme layer clean`.

- [ ] **Step 6: No academicpages remnants**

```bash
grep -rli "minimal-mistakes\|academicpages\|font-awesome\|academicons" _site/ && echo "FAIL" || echo "no theme remnants"
ls _site/assets/webfonts _site/assets/js 2>/dev/null && echo "FAIL: stale asset dirs" || echo "asset dirs clean"
```

- [ ] **Step 7: Screenshot all four pages in both schemes and look at every one**

```bash
for page in "" research teaching cv; do
  for scheme in light dark; do
    playwright screenshot --channel=chrome --color-scheme=$scheme --full-page \
      --viewport-size=1280,1000 "http://localhost:4000/$page" "/tmp/final-${page:-index}-$scheme.png"
  done
done
ls /tmp/final-*.png
md5 /tmp/final-index-light.png /tmp/final-index-dark.png   # must differ
```

Open all eight. Compare the light set against the static-site reference. Confirm in dark mode specifically: the paper is warm near-black, links are legible salmon, the nav rule and footer rule are visible but quiet, and `<details>` markers still read.

- [ ] **Step 8: Content completeness against the live site, all four pages**

```bash
for p in about research teaching cv; do curl -sS "https://sakshi-bhalla.github.io/$p/" -o "/tmp/live-$p.html"; done
python3 - <<'PY'
import re, html
def words(p):
    d = open(p, encoding='utf-8').read()
    d = re.sub(r'(?is)<(script|style|svg|head|nav|footer)\b.*?</\1>', ' ', d)
    d = re.sub(r'<[^>]+>', ' ', d)
    return set(re.findall(r"[A-Za-z][A-Za-z'&-]+", html.unescape(d).lower()))
for live, new in [('about', 'index'), ('research', 'research/index'),
                  ('teaching', 'teaching/index'), ('cv', 'cv/index')]:
    missing = words(f'/tmp/live-{live}.html') - words(f'_site/{new}.html')
    print(f"\n/{live}/ only-on-live:", sorted(missing))
PY
```

Every reported word must be attributable to a deliberate change listed in the spec's "Content model" section — the rewrites, the dropped sidebar and avatar, the dropped X link, or academicpages chrome. Anything else is an accidental loss and must be fixed before the PR.

- [ ] **Step 9: Stop the server and push**

```bash
pkill -f "jekyll serve"
git push -u origin redesign
```

- [ ] **Step 10: Open the PR**

```bash
gh pr create --base main --head redesign \
  --title "Rebuild the site on a slim custom Jekyll theme" \
  --body "$(cat <<'EOF'
Replaces the academicpages theme with a small self-authored one that reproduces
the static redesign, keeping Jekyll's templating.

## What changed

- **Theme:** one layout, four includes, one stylesheet — replacing ~50
  academicpages Sass partials, Font Awesome, Academicons, and the JS bundle.
- **Content:** publications move to `_data/publications.yml` rendered by an
  include; prose, news, and courses stay markdown. Page text comes from the
  static redesign, which reads as "PhD candidate", drops the "Prof." honorific,
  sets publication titles in sentence case, and corrects a news item to
  "National Institute on Drug Abuse".
- **Type:** Source Serif 4 and IBM Plex Sans are self-hosted (~107 KB, OFL
  licensed) instead of loaded from the Google Fonts CDN.
- **Dark mode** via `prefers-color-scheme`, contrast-checked to WCAG AA.
- **Deploy** moves from the classic Pages builder to GitHub Actions on Jekyll
  4.4.1, with html-proofer checking internal links on every build.
- **Removed:** four academicpages placeholder files in `files/` — a dummy bib
  entry and three "Slides 1/2/3" PDFs — that `sitemap.xml` was advertising to
  search engines.

## URLs

`/`, `/research/`, `/teaching/`, and `/cv/` all still resolve. `/blog/` and
`/feed.xml` now 404: the blog had no posts and its entire content was the string
"More soon."

## Action required before this deploys

**Settings → Pages → Source must be switched to "GitHub Actions".** Until then
the classic builder keeps serving the old site and the deploy job will fail.

## Verification

Clean build with no warnings; all routes return 200; html-proofer passes on
internal links; no `fonts.googleapis` reference survives; sitemap lists only real
pages and real PDFs; all four pages screenshotted in light and dark; and every
page diffed against the live site so nothing was dropped by accident.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

- [ ] **Step 11: Report the PR URL and the one manual step**

Tell the user the PR URL, and state plainly that the Pages source switch is theirs to make — the deploy cannot succeed without it.

---

## Follow-up work (not in this plan)

Extracting `_layouts/`, `_includes/`, and `assets/` into a standalone theme repo with demo content, a README, screenshots, and a `remote_theme:` consumption path. The no-hardcoding rule verified in Task 7 Step 5 is what keeps that extraction mechanical. It gets its own spec.
