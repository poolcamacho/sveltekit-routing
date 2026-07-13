# Routing Architecture Decisions

Each decision below is a real fork in a production SvelteKit routing design. For
each: why it exists, when to use it, when not to, the tradeoffs, alternatives, and
what migrating later costs. None of these has a universal winner. Present the
options and let the constraints decide.

## Table of contents
- [Flat vs nested route structures](#flat-vs-nested)
- [URL hierarchy vs internal code hierarchy](#url-vs-code)
- [Route groups vs physical nesting](#groups-vs-nesting)
- [Shared vs isolated layouts](#shared-vs-isolated)
- [Page routes vs API endpoints](#pages-vs-endpoints)
- [Server routes vs external backend](#server-vs-backend)
- [Route-level vs centralized authorization](#route-vs-central-authz)
- [Tenant id in hostname vs path vs session](#tenant-placement)
- [Locale prefixes vs locale negotiation](#locale)
- [Route-specific logic vs reusable services](#logic-vs-services)

## Flat vs nested route structures {#flat-vs-nested}

Why it exists: SvelteKit lets you express a resource either as one deep folder
chain or as a shallower set of routes. The choice trades layout sharing against
navigability.

- When nested: the URL genuinely has a hierarchy the product cares about
  (`/orgs/[org]/projects/[project]/settings`), and intermediate levels have their
  own pages and shared chrome.
- When flat: the segments do not each need a page or a layout, or the depth exists
  only to satisfy code organization. A flat tree is easier to scan and move.
- Tradeoffs: nesting gives free layout inheritance and clear parent-child data
  flow but multiplies layout loads and deepens paths; flat is simpler but may
  duplicate shell wiring.
- Alternatives: keep the URL shallow and express code grouping in `$lib` instead
  of in the route tree.
- Migration impact: flattening changes URLs and requires redirects; deepening is
  cheaper because you can add intermediate layouts without moving leaf URLs if you
  plan segment names up front.

## URL hierarchy vs internal code hierarchy {#url-vs-code}

Why it exists: the route tree encodes both at once, and they often want to differ.
The URL should reflect stable product concepts; the code wants cohesion and
ownership.

- Rule of thumb: design the URL from the product's public vocabulary first, then
  use route groups and layout resets to fit the code's needs without distorting
  the URL.
- When they align: small apps where the product hierarchy and the code grouping
  are the same shape. Do not manufacture a difference.
- When they diverge: an internal concern (which team owns a section, which layout
  a page uses) should not leak into the URL. Use groups (invisible in the URL) and
  keep domain code in `$lib`, imported by thin routes.
- Tradeoffs: forcing code structure into URLs produces unstable, refactor-coupled
  URLs; forcing URL structure into code produces awkward `$lib` layouts. Separating
  the two costs a little indirection.
- Migration impact: fixing a URL that mirrored code structure is a URL break with
  redirects. Fixing code structure behind stable URLs is free.

## Route groups vs physical nesting {#groups-vs-nesting}

Why it exists: both attach a shared layout to a set of routes; groups do it
without a URL segment, nesting does it with one.

- When groups: you want shared layout, a guard, or an error boundary for routes
  that must keep their existing URLs (an authed area at the root, distinct
  audiences on `/`).
- When physical nesting: the shared prefix is a real, meaningful URL segment
  (`/admin/...`), so the segment earns its place in the address.
- When neither: if there is no shared layout, guard, or boundary, do not create a
  group at all. A group with no shared behavior is cosmetic.
- Tradeoffs: groups keep URLs clean but are invisible in the address, so newcomers
  must learn the convention; nesting is self-documenting in the URL but forces a
  segment onto users.
- Migration impact: converting a physical segment into a group is a URL break;
  converting a group into a segment is also a URL break. Choosing correctly early
  avoids both.

## Shared vs isolated layouts {#shared-vs-isolated}

Why it exists: a layout can serve a whole subtree (shared) or a single route can
opt out with `@` resets (isolated).

- When shared: routes in a section have common chrome and common shell data
  (nav, session). Sharing is the default and the point of nested layouts.
- When isolated: a route inside a section for URL reasons must not inherit its
  chrome (full-screen editor, checkout, print view, embed). Use `+page@` or
  `+layout@`.
- Tradeoffs: over-sharing pushes feature data onto every child and couples the
  layout to a feature; over-isolating (resetting everywhere) means the layout tree
  no longer reflects the product and duplicates shell code.
- Alternatives: split a broad layout into a thin shell layout plus a nested
  feature layout, so only the feature subtree pays for feature data.
- Migration impact: introducing a reset is local and cheap; restructuring layouts
  can change which loads run on which navigations, so re-check data dependencies.

## Page routes vs API endpoints {#pages-vs-endpoints}

Why it exists: the same server capability can be exposed as a page with a form
action or as a `+server.ts` endpoint.

- When page + form action: the consumer is the app's own UI and you want
  progressive enhancement and typed load/action flow. This is the default for
  in-app mutations.
- When `+server.ts` endpoint: the consumer is a machine (mobile app, webhook,
  third party, another service), or you need a non-HTML response, custom methods,
  or streaming.
- Tradeoffs: form actions give co-located, progressively enhanced UX but are
  awkward for non-browser clients; endpoints are general but you give up the built
  in form ergonomics and must handle validation and errors yourself.
- Alternatives: expose a service in `$lib/server/` and call it from both a form
  action and an endpoint, so the capability has one implementation and two
  surfaces.
- Migration impact: moving a form action to an endpoint (or adding an endpoint
  alongside) is additive; removing a page URL that clients bookmarked is a break.

## Server routes vs external backend {#server-vs-backend}

Why it exists: SvelteKit can be the backend (`+server.ts`, server load) or a
frontend that calls a separate API service.

- When SvelteKit as backend: the app owns its data and logic, the team is
  full-stack, and you want one deployable with the security boundary built in.
- When external backend: an existing API or a polyglot backend already owns the
  domain, multiple frontends share it, or organizational boundaries put the API in
  another team or repo.
- Tradeoffs: internal endpoints mean one codebase, shared types, and no extra
  network hop, but couple frontend and backend deploys; an external backend
  decouples them at the cost of a network boundary, duplicated types, and two
  auth stories.
- Alternatives: hybrid, where server load and a thin BFF layer in SvelteKit call
  the external API, keeping secrets server-side and giving the client a tailored
  surface (see `examples/frontend-separate-backend.md`).
- Migration impact: moving from internal endpoints to an external API (or the
  reverse) changes where load fetches from; keeping load functions as the single
  data-access seam makes the swap local.

## Route-level vs centralized authorization {#route-vs-central-authz}

Why it exists: guards can live per route (each `+*.server.ts` checks) or be
centralized (hooks plus a layout guard for a subtree).

- When centralized: a whole section shares one boundary (everything under the
  authed group requires a session). Enforce it once in `hooks.server.ts` and/or
  the section's `+layout.server.ts` so no route can forget.
- When route-level: fine-grained authorization that varies per route (this page
  needs the `billing:admin` permission, that one does not). Check in the specific
  `+page.server.ts` or endpoint.
- Tradeoffs: centralized guards prevent the "forgot to guard a new route" bug but
  can be too coarse; per-route guards are precise but duplicate easily and are
  easy to omit on a new route.
- Best practice: a coarse server-side gate for the section (authentication and
  tenancy) plus per-route authorization for specific permissions, both on the
  server. Never rely on hiding a link. See `references/auth-and-boundaries.md`.
- Migration impact: consolidating scattered guards into a hook is a safe, testable
  refactor; loosening a central guard to per-route needs careful review so nothing
  becomes unguarded.

## Tenant id in hostname vs path vs session {#tenant-placement}

Why it exists: multi-tenant apps must locate the tenant on every request; the
three placements have different UX, isolation, and routing consequences.

- Hostname (`acme.app.com`): strong brand and isolation, natural per-tenant
  cookies, but needs wildcard DNS and TLS and adapter support for host routing.
  Resolve the tenant in `hooks.server.ts` from the host.
- Path (`/t/acme/...` or `/acme/...`): simplest to deploy (one domain), tenant is
  visible and linkable, but every route sits under a tenant segment and you must
  guard cross-tenant access on the server for each request.
- Session (tenant chosen after login, held in the session): cleanest URLs, but
  URLs are not shareable across tenants and switching tenants is stateful; risky
  if a user belongs to several tenants.
- Tradeoffs and alternatives: many products combine path-based tenancy with
  session-scoped defaults. Whatever the placement, the tenant boundary must be
  enforced in server code, never merely reflected in the URL.
- Migration impact: changing tenant placement changes every tenant-scoped URL and
  the auth flow; it is one of the most expensive routing migrations, so decide
  early. See `examples/multi-tenant-saas.md`.

## Locale prefixes vs locale negotiation {#locale}

Why it exists: localized apps either put the locale in the URL (`/en/...`,
`/fr/...`) or infer it (header, cookie, geolocation) and keep one URL.

- When prefixes: content should be indexable per language, shareable with an
  explicit locale, and SEO matters. Use `[[lang]]` optional or `[lang]` required
  with a matcher, and emit `hreflang` and canonical tags.
- When negotiation: an authenticated app where SEO is irrelevant and the user's
  preference is stored; keep URLs locale-free and switch content by session.
- Tradeoffs: prefixes give correct SEO and shareability but multiply the URL space
  and need redirect and default-locale handling; negotiation keeps URLs simple but
  hides language from links and crawlers and can serve the wrong language on cold
  requests.
- Alternatives: domain-per-locale (`example.fr`) where brand or legal reasons
  require it; more DNS and adapter work.
- Migration impact: adding a locale prefix later changes every content URL and
  needs redirects and canonical updates; starting with an optional `[[lang]]`
  segment leaves room without an immediate break. See
  `examples/localized-public-website.md`.

## Route-specific logic vs reusable services {#logic-vs-services}

Why it exists: the work a route does can live inline in the route file or in a
service the route calls.

- When inline: trivial, one-off, presentation-only work with no reuse and no test
  value. Do not over-abstract a three-line load.
- When a service: domain logic, I/O, validation, or anything used by more than one
  route or by both a form action and an endpoint. Put it in `$lib/server/`
  (server-only) or `$lib/` (shared) and keep the route thin: validate, call, shape.
- Tradeoffs: inline is fastest to write but traps logic in the URL layer, blocks
  reuse, and forces a request to test it; services cost a little ceremony but make
  logic testable, reusable, and safe to move when URLs change.
- Alternatives: start inline, extract on the second consumer or the first test.
- Migration impact: extracting inline logic to a service is a safe, incremental
  refactor done one route at a time; it also decouples the logic from URL changes,
  which is why it pays off before a routing migration.
