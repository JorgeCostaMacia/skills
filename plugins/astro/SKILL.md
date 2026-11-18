---
name: astro
description: >-
  How a static website is built with Astro and served as plain files. Use
  when creating or maintaining an Astro site, or when touching its
  astro.config, URL shape, styles, client scripts, Content Security Policy,
  configuration variables, fonts, images, metadata and structured data,
  sitemap, languages, content collections and their links, the web
  server's rules in front of it, or its go-live. It encodes: the
  configuration from the first commit, static output with clean URLs and
  real redirects in the server, scoped CSS that never wins by order and
  stays in cacheable files, behaviour as custom elements, a strict
  hash-based CSP with third-party hosts added per environment variable,
  build-time configuration with a hidden review build, self-hosted fonts,
  images sized to their boxes, per-page SEO and JSON-LD, prefixed locales
  with hreflang, content link tests, the server's headers and caching
  whatever the server, and testing on the real domain before the DNS moves.
---

# Static sites with Astro

A static Astro site: HTML, CSS, JS and images built once and served as plain files by a web server, with no server code at runtime. Which server is the project's choice; this skill says what it must do, not how a given one is configured. A site that needs a server at runtime (SSR, Actions, endpoints) is not this skill's. The toolchain is the `pnpm` skill's, tests the `testing` skill's and the docs `code-style-md`'s.

## Starting a site

- Start the repo with the configuration below from the first commit: retrofitting the URL shape or the CSP later touches every page.
- Pages are `.astro` files composed of small components, the text in the page.
- Content that people other than developers write (posts, news) is a Markdown content collection whose schema validates the front matter at build time, with a template file to start from.

## Output and URLs

- `trailingSlash: 'never'` and `build.format: 'file'`: every page is `route.html` and the server answers `/route` with it. In this mode `Astro.url.pathname` ends in `.html` (`/index.html` for the home): strip `(/index)?\.html` before building a canonical URL or comparing routes.
- Astro's `redirects` only write meta-refresh pages in static output. Every redirect is a 301 in the server, checked with `curl`; a meta-refresh page is at most a fallback behind it.
- One host is canonical (`www` or the apex) and the other answers a 301 to it.
- A URL that changes keeps a 301 from the old one.

## CSS

- Every component carries its own scoped `<style>`, with `scopedStyleStrategy: 'where'`: a scoped rule adds no specificity, so components override each other by their selectors, never by the order the stylesheets arrive in. The reset and the base styles are wrapped in `:where()` for the same reason; a utility class that must win says so with its own specificity.
- `build.inlineStylesheets: 'never'`: CSS is served as files, cacheable across pages, easy to debug, and the CSP carries no style hashes. A "render-blocking CSS" warning from PageSpeed is accepted; do not propose inlining it.
- Global values are custom properties in one tokens file; a variant is a class or a `data-*` attribute that sets custom properties, never `define:vars` (it writes a `style` attribute the CSP blocks).
- `vite.build.assetsInlineLimit: 0`: small images referenced from CSS stay files instead of `data:` URIs.

## Scripts

- Client behaviour lives in custom elements (`class X extends HTMLElement`, `customElements.define('x-y', X)`) defined in a plain `<script>` next to their markup: Astro processes it (TypeScript, bundled, deduplicated, hashed by the CSP). Server data reaches it through `data-*` attributes.
- No inline code. `is:inline` is for an external script whose host is in `script-src` and that must load as written, and for a JSON-LD data block (`type="application/ld+json"`, with `<` escaped as `<`), which the browser does not execute.
- No `<ClientRouter />` (it needs inline scripts); CSS cross-document view transitions need no JS.
- `markdown.syntaxHighlight: false` unless the site shows code: Shiki writes inline styles.

## Content Security Policy

- Astro's `security.csp` writes a `<meta>` with the hashes of its own scripts and styles. The directives: `default-src 'self'`, `base-uri 'self'`, `object-src 'none'`, `form-action 'self'`, `upgrade-insecure-requests`, `font-src 'self'`, and per directive only the origins the site loads. Never `'unsafe-inline'`: adding it to a directive also stops Astro emitting hashes for it.
- A third party's origins are added only when the variable that enables it is set: a small helper over `[enabled, origins]` pairs in `astro.config.mjs`. A build without the variable loads nothing of that third party and allows nothing of it.
- What a `<meta>` cannot carry (`frame-ancestors`, `report-to`, `sandbox`) is a header from the server; the browser enforces both.
- Check the built site in a browser for CSP violations on every page: the dev server applies no CSP.

## Configuration

- Everything is resolved at build time and every variable ends up in the HTML: a client variable is declared in `env.schema` with `access: 'public'` (or named `PUBLIC_*`), and read from `astro:env/client`. A site holds no secrets.
- The production values are committed in `.env.production`; `.env` is local and ignored by git.
- `astro:env` cannot be read in `astro.config.mjs`: load the files there with `process.loadEnvFile`, `.env.production` first and only for a build, then `.env`; it never overrides a variable already set, so the process environment wins.
- A hidden review build (`PUBLIC_NOINDEX=true`) for the review host before go-live: every page `noindex`, `robots.txt` disallowing everything, no sitemap, no `llms.txt`, no third party. It is removed from the production build at go-live.

## Fonts

Fonts are self-hosted through Astro's Fonts API with the local provider: open fonts from their Fontsource package files, commercial ones as `woff2` in `src/assets/fonts/`. Never a font CDN: it adds a third party and its CSP hosts. Preload only the fonts above the fold.

## Images

- Images live in `src/assets/` and go through `astro:assets` (WebP, resized, hashed). Content images are referenced by a stable path that a small resolver maps through an eager `import.meta.glob`, so an unknown path fails the build. `public/` keeps only what needs a fixed URL (the root favicon, `robots.txt`, linked PDFs).
- Generate at most about twice the largest box an image is shown in, measured at every width.
- Every image carries `width` and `height`, SVG included, with CSS `height: auto`; the LCP image gets `priority`; the rest load lazily.
- An animated GIF becomes an animated WebP (sharp with `animated: true`; Astro keeps the frames).
- A replaced original kept for later is renamed `name_old.ext` and excluded from the glob (`'!/src/assets/img/**/*_old.*'`).
- Read an imported image's properties from its `clone` (as `getImage` does): reading `image.width` marks the original as used and keeps it in `dist/`.
- A content image with a fixed width gets `max-width: 100%; object-fit: contain`.
- After a change to images, measure every image box of every page at 375, 768 and 1440 px against the previous build: the layout must not move.

## SEO and structured data

- Every page: a unique title, a meta description, a canonical URL, Open Graph and Twitter metadata with a preview image (through `getImage`), `og:type: article` on a post.
- JSON-LD on every page with the organization's most specific schema.org type, its address, geo position, phone and social profiles, a `logo` search engines can show (a PNG icon, not an SVG and not white on transparent) and an `image`, a photo of the place. A post adds `BlogPosting`. Check it with Google's Rich Results Test.
- `@astrojs/sitemap` with every indexable page (`noindex` pages filtered out), `robots.txt` pointing at it, a 404 page with `noindex`.
- An `llms.txt` and `llms-full.txt` written after the build from the built HTML, skipping `noindex` pages.
- Phone links as `tel:` with the country code; nothing that is plain text rendered as a link.

## Accessibility

One `h1` per page, heading levels in order, no empty heading; `header`, `nav`, `main`, `aside` and `footer` landmarks; a skip link; a visible `:focus-visible`; an accessible name on every icon-only link and button; menus usable with the keyboard; a dialog that moves and traps focus; form errors announced; motion off under `prefers-reduced-motion`. An axe audit of every page at a phone and a desktop width (WCAG 2.2 AA rules) reports nothing but the rules accepted on purpose, each listed with its reason in the audit script.

## Several languages

- Astro's i18n routing with every locale prefixed (`prefixDefaultLocale: true`) and `redirectToDefaultLocale: false`; the server answers `/` with a 301 to the default locale, and an Astro `redirects` entry is only its fallback.
- One table of locale tags feeds `<html lang>`, `og:locale` and `hreflang`.
- Every page links its translations with `hreflang`, plus `x-default`; the sitemap carries the same alternates. `@astrojs/sitemap`'s i18n option only pairs identical paths: translated slugs need its `serialize` hook.
- A language switcher links the same page in each language, or that language's home when the page does not exist in it.

## Content and its links

Tests over the content keep it sound: every internal link points to a page that exists, no link is relative (`[x](slug)` resolves against the current page), headings are in order and none is empty.

## The web server

Whatever serves the files:

- `/route` is answered from `route.html`; a missing path with `404.html` and status 404; any dotfile with 404, except `.well-known`.
- Headers on every response: `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Content-Security-Policy: frame-ancestors` with `X-Frame-Options`, `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy`, `Cross-Origin-Opener-Policy: same-origin`, defined once. A rule that sets headers of its own (a cache rule, say) must still send them: some servers drop the inherited headers there.
- Never an `X-Robots-Tag` on a public site. A host that served a hidden review build may still carry `noindex, nofollow`; it would deindex the site the day DNS moves.
- `/_astro/` (hashed) cached for good as `immutable`; HTML and any file whose name does not change revalidated with `no-cache`.
- A copy of the server's configuration in the repo is a reference; the server's is what answers. Before every go-live, check it header by header with `curl -sI`.

## Going live

1. The production build without the hidden review switch: `robots.txt` allows, the sitemap and `llms.txt` are generated, no robots meta but the 404's.
2. Every host the site uses (both hosts of the site, any API it calls) exists in the DNS zone and points at the right server.
3. Test the new server on the real domain before switching: `curl --resolve <host>:443:<ip>`, and Chrome with `--host-resolver-rules="MAP <host> <ip>"`. Crawl every sitemap URL (fetch the sitemap itself with `curl --resolve`: Playwright's `request` ignores the resolver rules): status, console errors, CSP violations, sideways scroll at 375 px, broken links.
4. Switch DNS, then repeat the crawl.
5. In Search Console: submit `sitemap-index.xml` and request indexing of the home and the key pages.
