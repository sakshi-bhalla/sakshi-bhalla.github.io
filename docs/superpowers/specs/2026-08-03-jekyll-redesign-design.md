# Jekyll redesign: slim theme from the static-site design

Date: 2026-08-03
Branch: `redesign` → PR into `main`

## Background

Commit `a19cc38` ("Replace Jekyll theme with static site") deleted the entire
academicpages theme and added nothing. It was an `--amend` of `d50dbb8`, and the
amend dropped the payload: `sakshi-site-redesign.zip`, a four-file static site
(`index.html`, `research.html`, `teaching.html`, `style.css`). That commit is
unreachable and would be destroyed by `git gc`, so it is preserved under the tag
`static-site-reference`.

The static site is the design reference. The decision is to keep Jekyll's
templating rather than ship flat HTML, and to rebuild the theme rather than
restyle academicpages — the design has no sidebar, avatar, breadcrumbs, or Font
Awesome, which is most of what academicpages' ~50 Sass partials exist to style.

## Goals

1. A Jekyll site that renders the static design faithfully.
2. Content edited as markdown and YAML, not HTML.
3. Local preview that matches production.
4. Deploy on push to `main` via GitHub Actions.
5. A theme layer clean enough to extract into a standalone theme repo without
   rewriting it (see "Theme extractability").

Non-goals: blog/`_posts`, 404 page, `jekyll-feed`, panel image, profile photo,
any part of academicpages. Extracting and publishing the theme repo itself is a
separate, later deliverable — this spec covers only the constraint that makes
that extraction cheap.

## Repository layout

```
.github/workflows/pages.yml
Gemfile
_config.yml
_data/navigation.yml
_data/publications.yml
_includes/head.html
_includes/masthead.html
_includes/footer.html
_includes/publication.html
_layouts/default.html
assets/css/style.css
assets/fonts/*.woff2
assets/fonts/OFL.txt
index.md
research.md
teaching.md
cv.md
files/          (existing, minus the placeholders below)
images/         (existing; only the favicon set and manifest.json are referenced)
```

`files/bibtex1.bib` and `files/slides1.pdf`, `slides2.pdf`, `slides3.pdf` are
deleted. All four are academicpages template placeholders — the bib entry is
"Alice, Bob and Charlie" in the *Journal of Examples*, and each slide deck is a
two-page PDF whose only text is "Slides 1/2/3". Nothing links them, but
`sitemap.xml` currently advertises all four to search engines. `cv.pdf` and
`paper1-3.pdf` are real and stay.

Pages live at the repository root. `_pages/` exists in academicpages only to work
around an `include:` quirk; four pages do not need it.

Everything from `main`'s `_layouts/`, `_includes/`, `_sass/`, `assets/js/`,
`assets/fonts/`, `assets/webfonts/`, `_data/{authors,cv,ui-text}.yml`,
`_data/comments/`, and `panel_img.png` is absent from the new tree.

## URLs

| Page     | URL           | Source        | In nav |
|----------|---------------|---------------|--------|
| About    | `/`           | `index.md`    | yes    |
| Research | `/research/`  | `research.md` | yes    |
| Teaching | `/teaching/`  | `teaching.md` | yes    |
| CV (PDF) | `/files/cv.pdf` | existing PDF | yes   |
| CV page  | `/cv/`        | `cv.md`       | no     |

Pretty permalinks, matching what `main` serves today — not the static site's
`/research.html`. Set per page via front-matter `permalink`.

The nav's CV entry points straight at the PDF, as the static design does — one
click to the document. `/cv/` is nonetheless rebuilt as a real page (its content
is the single "Download the full CV as PDF" line it has today) because that URL
is live and sitemapped; it stays reachable for old links and search results
without being duplicated in the nav.

`/blog/` is not rebuilt and will 404. It is live and sitemapped today, but its
entire content is the string "More soon." — there are no posts. `/feed.xml` will
also 404: `jekyll-feed` is dropped along with the blog, and the feed it currently
serves is empty.

## Content model

Hybrid: publications are structured data; prose, News, and Teaching are markdown.

### `_data/publications.yml`

One entry per paper:

```yaml
- title: "Partisan news users in the United States and India on either side seldom use fact checkers"
  authors: ["R. Ray", "S. Bhalla", "H. Taneja"]
  year: 2025
  venue: "Journal of Communication"
  status: published
  pdf: /files/paper1.pdf
  doi: https://academic.oup.com/joc/advance-article/doi/10.1093/joc/jqaf018/8129746
  abstract: >
    Fact checkers have low reach, and their limited efficacy is often attributed
    to perceived partisanship. ...
```

Fields: `title` (required), `status` (required: `published` | `forthcoming` |
`wip`), `authors` (required except for `wip`), `year` (required for `published`),
`venue`, `pdf`, `doi`, `abstract`, `note`. Optional fields are omitted entirely
when absent; the template must not emit empty elements for them.

The hanging date label is `year` for `published`, the literal `Forthcoming` for
`forthcoming`, and absent for `wip`. `note` renders in the byline slot for
entries that have no author list — it carries lines like "Earlier version
presented at the Local News Researchers Workshop, March 2024, Durham, NC."

Author bolding is not stored per entry. `_config.yml` holds
`author.short_name: "S. Bhalla"`; `_includes/publication.html` bolds whichever
author string matches it, so the name cannot fall out of sync across entries.
The template also renders the separators — comma-separated with `and` before the
last author, no comma for two authors.

Seed data is the four publications and two works-in-progress from
`static-site-reference:research.html`, which is itself the edited version of
`main:_pages/research.md`. The static HTML wins on wording where the two differ.

### `_data/navigation.yml`

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

### Pages

`index.md` carries the four about-page paragraphs plus a `## News` section of
hand-written `<div class="entry">` blocks (date span + prose), seven entries.
`teaching.md` is a markdown list of five courses. `research.md` is a lede
paragraph plus two loops over `site.data.publications` — `status != 'wip'` under
**Publications**, `status == 'wip'` under **Works in progress**. `cv.md` is one
line linking `/files/cv.pdf`.

Prose comes from `static-site-reference`, not from `main` — the static versions
are the rewritten ones, and their edits are deliberate, not drift. In particular
the static text says "PhD candidate" where the live site says "PhD student",
drops the honorific from "Prof. Scott Althaus", sets publication titles in
sentence case, and corrects the March 2025 news item to "National Institute on
Drug Abuse" (the live site's "Drug Addiction" is wrong — NIDA is Abuse). All of
these are kept.

Content on the live site that the new site deliberately does not carry: the
sidebar avatar, the panel image, the "Urbana, IL" location line (the role line
already names the university), and the X/Twitter link. The `twitter` key is
removed from `_config.yml` rather than special-cased in the template — the footer
include renders whichever social keys are present, so deleting the key is what
removes the link, and restoring it is a one-line change.

## Templating

`_layouts/default.html` is the only layout: `head` + `masthead` + `{{ content }}`
+ `footer`, wrapped in `<div class="page">`.

- `_includes/head.html` — meta, per-page `<title>` (`{{ page.title }} ·
  {{ site.title }}`, falling back to bare `{{ site.title }}` when the page has no
  title, as on the home page) and `description` from front matter,
  favicon set (`favicon.ico`, `favicon.svg`, `favicon-32x32.png`,
  `favicon-192x192.png`, `apple-touch-icon-180x180.png`, `manifest.json` — all
  already in `images/`), stylesheet, GA4 snippet, `google_site_verification`.
- `_includes/masthead.html` — name, role line, and nav from
  `_data/navigation.yml`. `aria-current="page"` is set by comparing `page.url` to
  each item's `url`, so the active-page highlight maintains itself.
- `_includes/footer.html` — social links generated from the `author:` block in
  `_config.yml` (email, googlescholar, github, bluesky, orcid, linkedin). A link
  is emitted only if its config key is set; adding one is a single YAML line.
- `_includes/publication.html` — renders one entry from `_data/publications.yml`.

Carried over from `main`'s `_config.yml`: `title`, `name`, `url`, `repository`,
`google_site_verification`, the `author:` block, and the GA4 tracking id
`G-CC9LEWNPER`. Dropping the verification token would risk the site's Search
Console status, so it stays.

## Styling

`assets/css/style.css` is `static-site-reference:style.css`, ported with three
changes. Plain CSS, not Sass — the design uses CSS custom properties and needs
no preprocessing.

Preserved exactly: `--ink #1a1a1a`, `--paper #fffefc`, `--accent #8a2f22`,
`--muted #6f6a64`, `--rule #e7e2da`; `42rem` column, `18px` root, `1.6` line
height; Source Serif 4 for text and IBM Plex Sans for nav, dates, pub links, and
footer; the `min-width: 62rem` rule that hangs entry dates in the left margin;
`<details>` abstracts with the `+` / `−` marker; `::selection`, `:focus-visible`,
and the `prefers-reduced-motion` block.

Changes:

1. **Self-hosted fonts.** Source Serif 4 (400, 600, 400 italic) and IBM Plex Sans
   (400, 500), latin subset, woff2, `font-display: swap`, served from
   `assets/fonts/`. Both are SIL Open Font License; `OFL.txt` ships alongside.
   The Google Fonts `<link>` and both `preconnect` hints are removed — no
   third-party request remains for fonts.
2. **Dark mode.** A `@media (prefers-color-scheme: dark)` block that redefines
   the `:root` custom properties only; no rule is duplicated. `#8a2f22` on a
   near-black background fails WCAG AA, so the dark accent is a lighter tint of
   the same madder hue. Every dark-mode pair — body text, muted text, accent
   links, link underline, rule — must be verified at ≥ 4.5:1 (≥ 3:1 for the
   non-text rule) before the work is called done.
3. **`img.panel` removed.** Styled in the static CSS but used by no page; the
   design has no panel image.

## Theme extractability

The theme is intended to ship later as a standalone repo, consumed via
`remote_theme:`. To keep that extraction from becoming a rewrite, one rule holds
throughout this work:

> **No file under `_layouts/`, `_includes/`, or `assets/` contains content
> specific to this person or site.**

Every name, role line, email, social URL, tracking id, nav item, page title, and
publication is read from `_config.yml`, `_data/`, or page front matter. This is
also why author bolding resolves against `author.short_name` rather than being
hardcoded, why footer links are generated from the `author:` block instead of
written out, and why nav comes from `_data/navigation.yml`.

Consequences the templates must handle, since a reuser will not have the same
data as this site:

- Any optional config key absent → the corresponding markup is not emitted at
  all (no empty `<p class="role">`, no bare `mailto:`, no GA4 snippet without a
  tracking id, no verification meta without a token).
- `_data/publications.yml` absent or empty → `research.md`'s loops render
  nothing rather than erroring.
- All internal URLs pass through `relative_url` so a project-site `baseurl`
  works, even though this site's `baseurl` is `""`.

Verification for this goal is item 8 below. The extraction itself — demo
content, README, screenshots, `remote_theme` wiring, license — is out of scope
here and gets its own spec.

## Build and deploy

`_config.yml` must `exclude:` this `docs/` directory. Markdown files without
front matter are otherwise copied verbatim into `_site`, which would publish this
spec at `sakshi-bhalla.github.io/docs/superpowers/specs/`.

`Gemfile` pins Jekyll 4.x plus `jekyll-sitemap` and `webrick` (Ruby 3 dropped
webrick from stdlib, and Jekyll 4's serve command needs it). Exact versions are
checked against rubygems.org at implementation time rather than assumed here.

Local: `brew install ruby`, then `bundle install` and `bundle exec jekyll serve`.
macOS stock Ruby is 2.6, which Jekyll 4 does not support, so the Homebrew Ruby is
required.

`Gemfile.lock` is committed, and the existing `.gitignore` line that excludes it
is removed. The workflow's `bundler-cache: true` keys its cache off the lockfile,
and an uncommitted lockfile means CI resolves gems fresh on every run and can
silently build with different versions than local. The same `.gitignore` pass
drops the stanzas for `.sass-cache/`, `/vendor/`, `.bundle/`, `node_modules`, and
`package-lock.json`, none of which apply to the new tree; `_site/` stays.

`.github/workflows/pages.yml`, triggered on push to `main` and
`workflow_dispatch`:

- `actions/checkout@v7` → `ruby/setup-ruby@v1` (with `bundler-cache: true`) →
  `actions/configure-pages@v6` → `bundle exec jekyll build` →
  `actions/upload-pages-artifact@v5` → `actions/deploy-pages@v5`
- permissions `contents: read`, `pages: write`, `id-token: write`
- concurrency group `pages`, `cancel-in-progress: false`
- Versions above are the latest majors as of 2026-08-03, read from each action's
  releases rather than copied from a template. Gem versions likewise: Jekyll
  4.4.1, `jekyll-sitemap` 1.4.0, `webrick` 1.9.2.

`html-proofer` runs in the same job and in local verification, checking that
every internal link and asset reference in `_site` resolves. External links are
disabled there — a journal being briefly down should not fail a deploy.

**Manual step, outside this work:** Settings → Pages → Source must be switched
to "GitHub Actions". Until then the classic builder keeps serving the old site
and the workflow's deploy step fails.

## Verification

All of the following must pass before the work is called done:

1. `bundle exec jekyll build` completes with no warnings.
2. Local server returns 200 for `/`, `/research/`, `/teaching/`, `/cv/`,
   `/files/cv.pdf`, `/assets/css/style.css`, and every file in `assets/fonts/`.
   `sitemap.xml` lists no placeholder file and no page that does not exist.
3. Headless-Chrome screenshots of all four pages in light and dark mode, each
   inspected; light mode compared against the static-site reference screenshots.
4. `grep -r fonts.googleapis _site` returns nothing.
5. Every internal `href` and `src` in `_site` resolves to a file that exists.
6. Dark-mode contrast ratios computed and confirmed against WCAG AA.
7. `_site` contains no file inherited from academicpages.
8. Theme extractability: grepping `_layouts/`, `_includes/`, and `assets/` for
   `Sakshi`, `Bhalla`, `illinois`, `sakshib3`, `G-CC9LEWNPER`, and the social
   handles returns nothing. Additionally, building with `_data/publications.yml`
   emptied and the optional `author:` keys removed still produces all four
   pages without errors or empty stub markup.
9. Content completeness: the text of every live page (`/about/`, `/research/`,
   `/teaching/`, `/cv/`) is diffed against the new build, and every difference
   is one of the deliberate drops or rewrites named in "Content model". Nothing
   is lost by accident.

## Risks

- **Pages source not flipped.** The deploy fails loudly on `deploy-pages`, not
  silently; the old site stays up until it is flipped. Called out in the PR body.
- **Font licensing.** Both families are OFL, which permits redistribution;
  `OFL.txt` must ship or the redistribution is non-compliant.
- **Content drift.** `main`'s markdown and the static HTML disagree in wording.
  The static HTML is authoritative; every page's prose is taken from there.
