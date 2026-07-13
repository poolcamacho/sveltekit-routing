# Routing Mechanics

The load-bearing rules of SvelteKit routing, with the edge cases that drive
architecture decisions. Confirm specifics against the current docs before giving
version-sensitive advice: https://svelte.dev/docs/kit/routing and
https://svelte.dev/docs/kit/advanced-routing.

## Table of contents
- [Special files](#special-files)
- [Dynamic, optional, and rest parameters](#parameters)
- [Parameter matchers](#matchers)
- [Route groups](#groups)
- [Nested layouts and inheritance](#layouts)
- [Layout resets: +layout@ and +page@](#resets)
- [Error routes](#errors)
- [API endpoints and HTTP methods](#endpoints)
- [Route precedence and conflicts](#precedence)
- [Trailing slash](#trailing-slash)
- [Prerender, SSR, CSR](#rendering)
- [Navigation and redirects](#navigation)
- [Adapter considerations](#adapters)

## Special files {#special-files}

A route directory can hold these files. Each has a distinct job; conflating them
is the root of most fat-route-file problems.

- `+page.svelte`: the page component. Presentational; receives `data` from load.
- `+page.ts` / `+page.js`: universal `load`. Runs on server for the first render
  and in the browser on client navigation. No secrets, no direct DB access.
- `+page.server.ts` / `.js`: server-only `load` plus form `actions`. The place for
  DB access, secrets, and per-request work that must not reach the client.
- `+layout.svelte`: wraps a page and its descendants. Renders `{@render children()}`.
- `+layout.ts` / `+layout.server.ts`: load data for a subtree. Runs on every
  navigation within that subtree, so keep the payload shell-wide.
- `+server.ts`: an API endpoint. Exports functions named for HTTP methods
  (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`, `HEAD`).
- `+error.svelte`: the error boundary for a subtree.

Server (`+*.server.ts`) code is a hard boundary enforced by the build. Anything
importing `$lib/server/` or a private `$env` module cannot end up in the client
bundle. Put auth and tenancy decisions on that side.

## Dynamic, optional, and rest parameters {#parameters}

- `[slug]`: a required dynamic segment. `/blog/[slug]` matches `/blog/hello` and
  exposes `params.slug`.
- `[[lang]]`: an optional segment. `/[[lang]]/about` matches both `/about` and
  `/en/about`. Useful for optional locale prefixes and default-language routes.
- `[...rest]`: a rest (catch-all) segment capturing zero or more segments as a
  single string. `/files/[...path]` matches `/files/a/b/c` with `params.path =
  "a/b/c"`. Use it for genuinely open-ended hierarchies (docs trees, file paths),
  and pair it with a matcher or explicit validation so it does not silently
  swallow routes you meant to define.
- Multiple params in one segment are allowed, for example `[x]-[y]` or
  `[x].[y]`, which is how you encode composite identifiers or file extensions.

Always validate parameter values on the server before trusting them. A matcher
narrows the URL space; it is not a substitute for authorization or existence
checks in load.

## Parameter matchers {#matchers}

A matcher in `src/params/<name>.ts` exports `match(param) => boolean`, and a
segment references it as `[id=integer]`. Matchers do two jobs:

1. Reject URLs that cannot be valid before load runs (a non-numeric id, an
   unknown locale), producing a 404 instead of a failed load.
2. Disambiguate precedence. `[id=integer]` and `[slug]` in sibling positions let
   `/123` and `/hello` resolve to different routes deterministically.

When not to add one: for a value whose validity depends on the database (does
this user exist?), a matcher cannot answer that. Validate in load and throw
`error(404)`. Matchers are for shape, not existence.

## Route groups {#groups}

A folder wrapped in parentheses, `(app)/`, groups routes and lets them share a
layout without contributing a URL segment. `/(app)/dashboard` serves at
`/dashboard`.

Legitimate uses:

- Attach one layout and one server guard to a set of routes (an authenticated
  area) without a URL prefix.
- Separate audiences that share the root path but need different shells
  (`(marketing)` vs `(app)`).

Not a legitimate use: grouping purely to tidy the file explorer. A group that
adds no shared layout, guard, or error boundary is cosmetic and adds indirection
without meaning (see `references/anti-patterns.md`). A route can belong to only
one group at a given level, so overlapping concerns (auth AND locale AND version)
usually want a mix of groups, real segments, and hooks rather than nested groups.

## Nested layouts and inheritance {#layouts}

Layouts nest by directory depth. A page inherits every `+layout.svelte` from the
root down to its folder, and the corresponding `+layout.(server.)ts` loads
compose: each layout load's return is merged into `data` for descendants.

Consequences that matter for architecture:

- A layout load runs on every navigation within its subtree. Fetching a feature's
  detailed data in a broad layout adds latency to every child navigation.
- `depends`/`invalidate` and parent/child load ordering determine what re-runs.
  Keep layout loads to session, navigation, and other shell-wide needs.
- Shared vs isolated layouts is a real decision, covered in
  `references/architecture-decisions.md`.

## Layout resets: +layout@ and +page@ {#resets}

By default a page uses the nearest layout chain. A reset breaks out of it:

- `+page@.svelte` (bare `@`) resets to the root layout.
- `+page@(app).svelte` resets to the layout at the named segment or group.
- `+layout@.svelte` resets the layout subtree similarly.

Use resets when a route sits inside a section for URL reasons but must not inherit
that section's chrome: a full-screen editor inside the app shell, a checkout step
that hides navigation, a print view. Resets are the escape hatch that lets URL
hierarchy and layout hierarchy diverge on purpose. Overusing them is a smell that
the layout structure does not match the product; if you reset constantly,
reconsider where the layouts sit.

## Error routes {#errors}

`+error.svelte` renders when a `load` in its subtree throws. The nearest error
boundary up the tree handles the failure; without one, it bubbles to the root.
Place error boundaries where the surrounding chrome can still render (an app-shell
error page that keeps the nav) rather than only at the root. Expected failures
(not found, forbidden) should `throw error(404)` / `throw error(403)` from load
so the boundary can present them; unexpected throws become 500s and should be
handled by `handleError` in hooks for logging.

## API endpoints and HTTP methods {#endpoints}

`+server.ts` exports one function per HTTP method. Return a `Response` (often via
the `json` helper). Guidance:

- Keep handlers thin: parse and validate input, call a service, shape the
  response. Domain logic belongs in `$lib/server/`, not inline in the handler.
- Handle methods explicitly and let unlisted methods 405 rather than silently
  doing nothing.
- A directory cannot serve both a page and an endpoint for the same request in a
  way that is ambiguous; keep a URL either a page or an API, not both by accident.
- For content negotiation or form posts, prefer form actions on `+page.server.ts`
  for progressively-enhanced page forms, and reserve `+server.ts` for machine
  clients, webhooks, and non-page APIs.

## Route precedence and conflicts {#precedence}

When several routes could match a URL, SvelteKit sorts them by specificity.
Broadly: more specific (static) segments win over dynamic ones, matched dynamic
segments (`[id=integer]`) win over unmatched (`[slug]`), and rest segments
(`[...x]`) are least specific. Optional segments and multiple candidates can make
this subtle.

Practical rules:

- Do not rely on remembered ordering. When two routes could collide, add a matcher
  so precedence is explicit and deterministic, and confirm behavior against the
  current sorting rules in the advanced-routing docs.
- A `[...rest]` catch-all placed too high swallows sibling routes added later.
  Scope catch-alls to the narrowest subtree that needs them.
- If `/[slug]` and `/about` coexist, `/about` wins; but a new dynamic sibling can
  change which URLs a matcher-less segment absorbs. Prefer matchers over hope.

## Trailing slash {#trailing-slash}

`trailingSlash` is a page option (`'never'`, `'always'`, `'ignore'`) that cascades
to descendants. Inconsistent trailing-slash handling causes duplicate URLs (two
addresses for one page), redirect surprises, and canonical-URL and SEO problems.
Pick one policy for the app, set it high in the tree, and make canonical tags and
redirects agree with it. Some static hosts and adapters have their own trailing
behavior; verify against the adapter before assuming.

## Prerender, SSR, CSR {#rendering}

Three page options, cascading to children:

- `prerender = true`: build the route to static HTML. Valid only for pages that
  are the same for everyone and not per-request. Never prerender an authenticated
  or per-user route; you would ship one user's page to everyone or fail the build.
- `ssr = false`: skip server rendering; the route becomes a client-rendered shell.
  Costs SEO and first-paint content; acceptable for private app internals behind a
  login.
- `csr = false`: ship no client JavaScript for the route; fully static delivery of
  server-rendered HTML. Good for pure content pages; breaks client interactivity.

These are routing-architecture concerns because they cascade and because they
interact with auth (do not prerender protected routes) and locale (prerender the
localized variants you actually serve).

## Navigation and redirects {#navigation}

- Client navigation uses `<a>` links (enhanced automatically) and the programmatic
  `goto`. Prefer real anchors for accessibility and SSR.
- Redirect from load or actions with `throw redirect(status, location)`; use 307
  or 308 for temporary or permanent method-preserving redirects, 303 after a POST.
- Guard redirects against loops: a redirect target that itself redirects back
  (login redirects to app redirects to login) hangs the user. Make the guard
  idempotent and test the unauthenticated and authenticated paths.
- Invalid routes render the nearest `+error.svelte` as a 404. Provide a helpful
  root error page.

## Adapter considerations {#adapters}

The deployment adapter changes what routing features are available or cheap:

- Prerendering needs a host that serves static files; fully static adapters
  (`adapter-static`) require every route to be prerenderable or excluded.
- Edge and serverless adapters may split routes into functions, which affects cold
  starts and where server load runs; very large `+server.ts` files can inflate
  bundle size per function.
- Trailing-slash and redirect behavior can be enforced or overridden at the CDN or
  platform layer; confirm the adapter's docs so app-level `trailingSlash` and
  platform rules do not fight.
- Multi-region or edge deployments interact with tenancy-by-hostname and
  locale-by-domain strategies; verify the adapter supports the host routing you
  plan. Docs: https://svelte.dev/docs/kit/adapters.
