# Codebase notes

Hugo site for <https://crashdump.info/> (repo `raphideb/raphideb.github.io`), built with
the [Docsy](https://www.docsy.dev/) theme.

## Build & deploy

- **Theme install: Hugo Modules.** `go.mod` + `[module]` / `[[module.imports]]` in
  `hugo.toml`. There is **no** `package.json` and no npm-based theme install — keep it
  that way (see "Docsy version ceiling" below).
- **CI:** `.github/workflows/hugo.yml` — pushes to `main` build with `hugo --minify` and
  deploy the `public/` artifact to GitHub Pages via `actions/deploy-pages`. Hugo version
  is pinned in the workflow's `HUGO_VERSION` env var.
- The workflow `npm install`s `postcss postcss-cli autoprefixer` only because Docsy pipes
  the CSS through `postCSS` in production builds. That is not a theme dependency.
- Build output is **gitignored** (`/public/`, `/resources/_gen/`, `/.hugo_build.lock`).
  Both were tracked until the Docsy 0.15 push; since Hugo only overwrites the files it
  generates, the stale committed copies were being uploaded alongside the fresh build.
  The workflow also passes `--cleanDestinationDir` so the artifact can only ever contain
  what that run produced.

### Local builds

`hugo server` and a plain `hugo` write into `public/`, which is gitignored — harmless, but
to keep it out of the way entirely, build elsewhere:

```
hugo -e development --baseURL http://localhost:8099/ --destination /tmp/build
```

A plain `hugo` (production) build fails locally unless `postcss` is on `PATH` — use
`-e development`, or let CI do the production build.

## Docsy version ceiling

Pinned to **docsy v0.15.0**. This is deliberate — it is the last release that installs as
a plain Hugo module:

| | ≤ 0.15.0 | ≥ 0.16.0 |
|---|---|---|
| module path | `github.com/google/docsy` | `github.com/google/docsy/theme` |
| Bootstrap / Font Awesome | Hugo module imports | `node_modules/` only (`hugo mod npm pack` + `npm install`) |
| Sass | Hugo's embedded LibSass | 0.17 hardcodes `transpiler: dartsass`, needs a `sass` binary |

Moving to 0.16+ means adding Node.js and Dart Sass to the local toolchain.

## Project overrides

Docsy is customised only through project files that shadow theme files — no theme fork.

### `assets/scss/`

- `_variables_project.scss` — SCSS variables, imported by Docsy before its own
  `_variables.scss`. Currently pins `$td-navbar-bg-color: #30638e` (Docsy 0.14 changed
  the navbar default from the primary colour to the body background) and sets
  `$td-enable-google-fonts: true` (0.14 flipped it to false; without it the whole site,
  navbar brand included, falls back to the system font stack instead of Open Sans).
- `_styles_project.scss` — appended CSS. `@import 'td/code-dark'` enables dark code
  blocks; hides the Docsy "view/edit/new page" meta links; table and `.floatimg` styles;
  removes the navbar bottom border Docsy 0.14 added; and reproduces Docsy 0.12's navbar
  brand — 0.14+ made it a flex row (`display: flex; align-items: center; gap: .5rem`)
  with a 2rem logo and a 600-weight name, 0.12 used inline flow with a baseline-aligned
  30px logo and a bold name, which keeps `ok#` and `crashdump` on one line at one size.

Docsy defaults that were **not** pinned back on the 0.12 → 0.15 upgrade, so the site
picked up the new look: `$primary` (was `#30638e`, now Bootstrap blue), `$secondary`
(was `#ffa630`), Google Fonts off (Open Sans → system stack), and code highlighting
(tango/onedark → friendly/native, which makes `markup.highlight.style` a no-op).

### `assets/js/dark-mode.js`

Fork of Docsy's own file. Adds `data-theme-key` / `data-theme-default` support on
`<html>` so a page can use its own localStorage key and default theme. **When bumping
Docsy, re-fork from the new upstream file and re-apply those two hunks** — upstream
changes here (e.g. the 0.13 `removeAttribute('data-theme-init')` FOUC fix) are easy to
lose.

### `layouts/partials/`

- `hooks/head-end.html` — canonical link, Cloudflare Web Analytics beacon, and the
  **astronomy per-section dark mode**: pages under `/astronomy/` default to dark and
  remember their choice under a separate key (`td-color-theme-astronomy`), independent of
  the rest of the site. Works together with `assets/js/dark-mode.js`.
- `page-meta-lastmod.html` — simpler "last modified" line without the git commit link.
- `back-to-top.html` — **unused**, nothing references it.
- `sidebar.html` — copy of Docsy 0.15.0's with one changed line. Docsy ≤0.13 rooted the
  sidebar at the whole site tree when the home page is `type: docs`; from 0.14 the
  `sidebar_root` feature re-roots it at `.FirstSection`, which scopes the menu to the
  current section. The override restores the full tree (still needed in 0.17).
- `head-css.html` — copy of Docsy 0.15.0's, `.Site.Language.LanguageDirection` →
  `.Site.Language.Direction`. **Shim; delete on Docsy ≥0.16**, which fixed it upstream.

### `layouts/` (root)

- `baseof.html`, `docs/baseof.html` — same `LanguageDirection` shim as above. All content
  is `type: docs`, so `docs/baseof.html` covers every content page and the root
  `baseof.html` covers home, `/tags/` and 404.

### `layouts/shortcodes/`

- `gallery.html` — the astronomy image gallery (`/astronomy/images/`); generates webp
  thumbnails with `.Process "fit 640x640 webp q80"`.
- `xkcd.html` — embeds an xkcd strip (used on the home page).

### `assets/icons/logo.svg`

The `ok#` navbar logo, inlined by Docsy's navbar partial.

## Content

`content/` has four sections, all `type: docs`, plus a `type: docs` home page (that is
what makes the full sidebar tree possible): `astronomy`, `kubernetes`, `oracle`,
`postgres`. `public/stuff/` and `public/vibeshake.html` are stale output for content that
no longer exists.
