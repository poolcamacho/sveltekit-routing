# URL Design: SEO, Canonicals, Naming, Locales, and API Versioning

URLs are a public contract. This file covers SEO-friendly and canonical URLs,
route naming, locale-aware routes, and versioned API routes with a migration
story. The theme: choose URLs for durability and meaning, and treat any change as
a migration.

## Table of contents
- [Route naming](#naming)
- [SEO-friendly and canonical URLs](#seo-canonical)
- [Trailing slash as a canonical concern](#trailing-slash)
- [Locale-aware routes](#locales)
- [Versioned API routes](#api-versioning)
- [Invalid routes and not-found handling](#invalid)

## Route naming {#naming}

Name routes after durable product concepts, not code, UI, or implementation.

- Prefer nouns for resources (`/invoices`, `/invoices/[id]`) and clear verbs or
  sub-resources for actions (`/invoices/[id]/refund`).
- Use lowercase, hyphenated segments (`/billing-history`), consistently across the
  app. Pick one case convention for route folders and hold to it.
- Keep segments stable: do not encode a redesign, an experiment, or a component
  name in the path (see `references/anti-patterns.md`). Route experiments behind
  flags or query parameters.
- Do not leak internal identifiers. Use slugs or opaque ids in the URL and map to
  internal ids on the server.
- Map URL hierarchy to product hierarchy, and use route groups when the code needs
  grouping the URL should not show.

## SEO-friendly and canonical URLs {#seo-canonical}

For public, indexable pages:

- Give every indexable page one canonical URL and emit a `<link rel="canonical">`
  pointing at it. Decide canonical form for trailing slash, casing, and locale,
  and make redirects agree.
- Server-render indexable content (do not set `ssr = false` on pages that need to
  rank), and prerender static content where possible for speed.
- Reflect real hierarchy in the path so breadcrumbs and crawlers follow it, and
  keep slugs human-readable and stable. When a slug must change, 301-redirect the
  old one.
- Emit per-page titles and meta from load data, and provide a sitemap (often a
  `+server.ts` that serves XML) enumerating canonical URLs.
- Avoid duplicate content: one page should not be reachable at several
  canonical-looking URLs (trailing slash variants, tracking-param variants that
  render identically). Consolidate with canonical tags and redirects.

## Trailing slash as a canonical concern {#trailing-slash}

`trailingSlash` (`'never' | 'always' | 'ignore'`) is a page option that cascades.
Inconsistency produces two URLs for one page, splitting SEO signal and confusing
caches. Choose one policy, set it high in the tree, and align canonical tags and
redirects with it. Confirm the deployment adapter or CDN does not impose a
conflicting rule. See `references/routing-mechanics.md`.

## Locale-aware routes {#locales}

Two strategies (full tradeoffs in `references/architecture-decisions.md`):

- Locale prefixes in the URL, for indexable content:
  - Use an optional `[[lang]]` segment for a default-locale-at-root scheme, or a
    required `[lang=locale]` with a matcher listing supported locales.
  - Resolve and validate the locale in server load; redirect unknown or missing
    locales to a sensible default.
  - Emit `hreflang` alternates and a per-locale canonical so crawlers map the
    translations. Prerender the locale variants you actually serve.
  - Example shape: `/[[lang]]/products/[slug]` serving `/products/x` and
    `/fr/products/x`.
- Locale negotiation without URL prefixes, for authenticated apps:
  - Resolve locale from the session, cookie, or `Accept-Language` in
    `hooks.server.ts`, keep URLs locale-free, and switch content by preference.
  - Acceptable when SEO is irrelevant; not appropriate for public content that
    must be indexed and shared per language.

Whichever you choose, keep the locale resolution in one place (a matcher plus
server load, or a hook), not scattered per route.

## Versioned API routes {#api-versioning}

`+server.ts` endpoints that machine clients consume need a version and a plan for
change.

- Version at a deliberate boundary when you have external consumers you cannot
  update in lockstep. A path prefix (`/api/v1/...`) is explicit and cache-friendly;
  a header-based version is cleaner URLs but harder to inspect and cache. Choose
  based on who calls you.
- Share the implementation across versions: both versions' handlers call the same
  `$lib/server/` service, differing only in request parsing and response shaping.
  Do not fork the domain logic per version.
- Publish a deprecation and sunset policy: announce, set a date, add response
  headers or telemetry to observe remaining old-version traffic, and remove the
  version only when usage is safely low.
- Do not ship breaking changes into an unversioned endpoint; additive,
  backward-compatible changes are fine, breaking ones need a new version.
- Keep internal, first-party endpoints (consumed only by this app's own load and
  actions) unversioned if you deploy them together; version is for consumers you
  do not control.

## Invalid routes and not-found handling {#invalid}

- Unmatched URLs render the nearest `+error.svelte` as a 404. Provide a helpful
  root error page with navigation back into the app.
- Distinguish "route does not exist" (no matching file) from "resource does not
  exist" (route matches, load throws `error(404)`); both should present a clean
  not-found page, and the latter is where you avoid confirming existence to
  unauthorized callers by preferring 404 over 403.
- For moved URLs, prefer a real redirect (301/308) over a soft 404 so links and
  SEO transfer. Keep a redirects map for retired URLs rather than deleting them
  silently.
