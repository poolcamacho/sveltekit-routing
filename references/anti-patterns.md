# Routing Anti-patterns: Detection and Fixes

Routing problems that recur in SvelteKit codebases. Each entry covers what it
looks like, how to detect it, why it hurts, and how to fix it incrementally.
Frame these to the user as tendencies to correct, not failures. Most are the
natural result of a route tree growing faster than its design.

## Table of contents
- [Deep nesting without a product reason](#deep-nesting)
- [Layouts doing unrelated work](#fat-layouts)
- [Client-only navigation authorization](#client-auth)
- [Business logic inside route files](#logic-in-routes)
- [Duplicate route guards](#duplicate-guards)
- [Route groups as cosmetic folders](#cosmetic-groups)
- [Unstable URL design](#unstable-urls)
- [Leaking internal database identifiers](#leaked-ids)
- [Excessive catch-all routes](#catch-all)
- [Mixing page endpoints and domain services](#mixing-endpoints)
- [API versioning without a migration strategy](#versioning)
- [Route names tied to temporary UI](#ui-named-routes)
- [Redirect loops](#redirect-loops)
- [Inconsistent trailing slash](#trailing-slash)
- [Route code duplicated across branches](#duplicated-branches)
- [Large +page.server.ts / +server.ts](#large-files)
- [Missing parameter validation](#missing-validation)
- [Incorrect route-precedence assumptions](#precedence)
- [Reliance on deprecated routing behavior](#deprecated)

## Deep nesting without a product reason {#deep-nesting}
**Looks like.** `routes/app/dashboard/reports/monthly/detail/[id]/view/+page.svelte`
where most levels have no page and no layout of their own.
**Detect.** Route paths four or more segments deep where intermediate folders
contain only a single child and no `+page`/`+layout`, and URLs users never see as
distinct pages.
**Why it hurts.** Long URLs, layout loads that run for empty levels, and paths
that are painful to move. Depth that does not reflect product hierarchy is pure
cost.
**Fix.** Collapse intermediate segments that carry no page or layout. Keep the
segments the product genuinely names, and express code grouping in `$lib` instead
of the route tree. Redirect any collapsed URL that shipped.

## Layouts doing unrelated work {#fat-layouts}
**Looks like.** A `+layout.server.ts` near the root fetching a feature's detailed
data (a full report, a product catalog) for the whole subtree.
**Detect.** Layout loads returning large or feature-specific payloads, or heavy
queries re-running on every child navigation within the subtree.
**Why it hurts.** Layout loads run on all descendant navigations, so over-fetching
there adds latency everywhere and couples the shell to one feature.
**Fix.** Keep layout loads to shell-wide needs (session, nav, tenant). Move
feature data into the specific `+page` load that needs it. If a subtree genuinely
shares feature data, give it its own nested layout rather than loading it at the
root.

## Client-only navigation authorization {#client-auth}
**Looks like.** A protected page that is guarded only by hiding its link, an
`{#if user.isAdmin}` in a component, or a check in universal `+page.ts`.
**Detect.** A route with sensitive data whose only guard is in `.svelte` or in a
universal load; no check in `+*.server.ts`, hooks, or the endpoint. Try requesting
the URL directly while unauthorized.
**Why it hurts.** Anyone can type the URL or call the endpoint. Hiding a link is
not access control; universal load runs in the browser and cannot be trusted.
**Fix.** Enforce on the server: a session/authorization check in
`hooks.server.ts`, the section's `+layout.server.ts`, or the specific
`+page.server.ts`/`+server.ts`. Throw `error(401)`/`error(403)` or redirect. Keep
the client check only as UX, never as the boundary. See
`references/auth-and-boundaries.md`.

## Business logic inside route files {#logic-in-routes}
**Looks like.** A `+page.server.ts` of 150 lines computing pricing, applying
entitlement rules, and hitting three tables inline in `load` or an action.
**Detect.** Route files much larger than their siblings, domain vocabulary
implemented directly in load/actions, and no `$lib` import.
**Why it hurts.** Logic cannot be reused by another route or an endpoint, cannot
be tested without a request, and URL changes now risk domain code.
**Fix.** Extract domain rules into `$lib` (pure functions) and I/O into a
`$lib/server/` service. The route becomes: validate input, call service, shape
response. Do it one route at a time.

## Duplicate route guards {#duplicate-guards}
**Looks like.** The same session-and-role check copy-pasted at the top of ten
`+page.server.ts` files.
**Detect.** Identical guard blocks across route files, and a new route that
forgot to include one.
**Why it hurts.** Guards drift, and the guard you forget is the vulnerability.
Duplication here is a security risk, not just a style issue.
**Fix.** Lift the coarse boundary (authentication, tenancy) into
`hooks.server.ts` or a group's `+layout.server.ts`, so it applies to every route
under it. Keep only route-specific permission checks in the route, calling a
shared `requirePermission` helper. See `references/architecture-decisions.md`.

## Route groups as cosmetic folders {#cosmetic-groups}
**Looks like.** `(dashboard)`, `(stuff)`, `(misc)` groups that add no shared
layout, guard, or error boundary and exist only to tidy the explorer.
**Detect.** A `(group)/` folder with no `+layout*` and no shared server logic, or
several groups whose only difference is the folder name.
**Why it hurts.** Indirection with no meaning. Newcomers hunt for a layout that
does not exist, and the group implies a boundary that is not enforced.
**Fix.** Remove the parentheses (making it a real segment) if the prefix is
meaningful, or flatten the group away if it is not. Reserve groups for a genuine
shared layout, guard, or error boundary.

## Unstable URL design {#unstable-urls}
**Looks like.** URLs that change whenever the code or navigation is refactored,
or that encode implementation details (`/v2-new-dashboard-final/...`).
**Detect.** Frequent redirect churn in git history, URLs that name frameworks,
components, or sprint names, and links that break between releases.
**Why it hurts.** URLs are a public contract: bookmarks, links, and search index
entries rot. Every change costs redirects and lost SEO.
**Fix.** Name routes after durable product concepts, decouple URL structure from
code structure (groups, `$lib`), and treat a URL change as a migration with
redirects. See `references/url-design.md`.

## Leaking internal database identifiers {#leaked-ids}
**Looks like.** `/users/48213/orders/99a1f...` exposing auto-increment or raw
primary keys.
**Detect.** Route params that are database primary keys, sequential integers that
reveal record counts, or ids that let one user probe another's resources.
**Why it hurts.** Enumerable ids invite scraping and IDOR-style probing, leak
business metrics (how many users you have), and couple the URL to the storage
schema.
**Fix.** Use opaque public identifiers (slugs, UUIDs, or hashed ids) in URLs and
map to internal ids server-side. Regardless of id opacity, always authorize the
resource in server load; an unguessable id is not access control.

## Excessive catch-all routes {#catch-all}
**Looks like.** A `[...path]` near the root that handles many concerns with a big
`if/else`, or several overlapping catch-alls.
**Detect.** A rest segment high in the tree, a catch-all load with branching on
`params.path`, and sibling routes that mysteriously stop resolving after one was
added.
**Why it hurts.** A high catch-all swallows routes added later, hides real
structure, and turns routing into hand-rolled dispatch.
**Fix.** Define explicit routes for known cases and scope the catch-all to the
narrowest subtree that truly needs open-ended paths (a docs tree, a CMS slug
space). Add a matcher so the catch-all only accepts what it should.

## Mixing page endpoints and domain services {#mixing-endpoints}
**Looks like.** A `+server.ts` that both parses the request and implements the
domain logic, with other routes importing from that endpoint file.
**Detect.** Imports pointing at `+server.ts` or `+page.server.ts` from elsewhere,
and domain functions defined in route files.
**Why it hurts.** Route files are not a module API; importing across them couples
URLs to logic and breaks when routes move. The endpoint conflates transport with
domain.
**Fix.** Move domain logic into `$lib/server/` services. The endpoint and any form
action both call the service. Nothing imports from a `+` file.

## API versioning without a migration strategy {#versioning}
**Looks like.** `/api/v1/...` and `/api/v2/...` both live indefinitely with no
deprecation plan, or breaking changes shipped into `/api/...` with no version at
all.
**Detect.** Multiple version prefixes with duplicated handlers and no sunset
policy, or a single unversioned API whose response shape changed between releases.
**Why it hurts.** Either clients break silently (unversioned breaking changes) or
versions accumulate forever with duplicated, diverging code.
**Fix.** Version at a deliberate boundary, share the implementation across
versions via services, document a deprecation and sunset policy, and add
telemetry to know when an old version is safe to remove. See
`references/url-design.md`.

## Route names tied to temporary UI {#ui-named-routes}
**Looks like.** `/new-blue-dashboard`, `/beta`, `/tab2`, URLs named after a
button or a redesign.
**Detect.** Route folders named after UI states, experiments, or visual treatments
rather than resources.
**Why it hurts.** When the UI changes, the URL is either wrong or frozen as a lie.
Users and crawlers hold URLs long after the UI moves on.
**Fix.** Name routes after the resource or task (`/reports`, `/settings/billing`),
not its current presentation. Route experiments behind flags or query params, not
permanent path segments.

## Redirect loops {#redirect-loops}
**Looks like.** Login redirects to app, app guard redirects back to login, and the
browser hangs; or a trailing-slash redirect that bounces.
**Detect.** "Too many redirects" errors, and guards that redirect without checking
whether they are already at the target.
**Why it hurts.** The route becomes unreachable for the affected users.
**Fix.** Make guards idempotent: redirect only when the condition is unmet and the
target differs from the current path. Test both authenticated and unauthenticated
paths, and reconcile trailing-slash redirects with the `trailingSlash` policy.

## Inconsistent trailing slash {#trailing-slash}
**Looks like.** Some routes served at `/x`, some at `/x/`, canonical tags and
redirects disagreeing.
**Detect.** Mixed `trailingSlash` settings, duplicate URLs for one page in
analytics, and 301s that flip between forms.
**Why it hurts.** Duplicate URLs split SEO signals and confuse caching and
canonical tags.
**Fix.** Choose one policy (`'never'` is common), set it high in the tree, and
make canonical tags and redirects agree. Verify the adapter or CDN does not impose
a conflicting rule.

## Route code duplicated across branches {#duplicated-branches}
**Looks like.** The same load, guard, and data-shaping copy-pasted across
`(marketing)`, `(app)`, and `admin` versions of a similar page.
**Detect.** Grep for repeated load bodies or near-identical `+page.server.ts`
files in different branches.
**Why it hurts.** Fixes must land in several places and drift apart.
**Fix.** Extract the shared load logic into a `$lib` function the branches call,
or use a shared layout load where the data is genuinely subtree-wide. Extract only
what is truly the same concept, not merely similar today.

## Large +page.server.ts / +server.ts {#large-files}
**Looks like.** A single route file of many hundreds of lines with several
actions, inline validation, and inline queries.
**Detect.** Route files that are the largest in the repo, many exported actions or
methods in one file, and mixed concerns.
**Why it hurts.** Hard to read, hard to test, and a merge-conflict magnet;
serverless adapters may also inflate the per-function bundle.
**Fix.** Push validation into schemas, domain logic and I/O into `$lib/server/`
services, and keep the file to thin handlers. If one endpoint serves many
resources, consider splitting the routes.

## Missing parameter validation {#missing-validation}
**Looks like.** A route trusting `params.id` or `[...path]` directly in a query or
a filesystem read.
**Detect.** Params used in data access without a matcher and without a runtime
check, or catch-all values passed to `fetch`/`fs`/SQL.
**Why it hurts.** Malformed or hostile input reaches data access: failed loads,
injection surfaces, or path traversal via a rest segment.
**Fix.** Add a matcher for shape (integer, uuid, known enum) to reject bad URLs as
404s, and validate existence and authorization in server load. Never pass a raw
rest param to the filesystem without normalizing and constraining it.

## Incorrect route-precedence assumptions {#precedence}
**Looks like.** Assuming `/[slug]` will not shadow `/about`, or that a new dynamic
sibling will not change which URLs a matcher-less segment absorbs.
**Detect.** Routes that "sometimes" resolve to the wrong page, behavior that
changed after adding a sibling route, and no matchers disambiguating siblings.
**Why it hurts.** The wrong page or endpoint answers a URL, which is both a bug
and, if the wrong one is less guarded, a security issue.
**Fix.** Add matchers so precedence between static, matched-dynamic, unmatched
dynamic, and rest segments is explicit, and confirm behavior against the current
sorting rules in the advanced-routing docs. Do not rely on remembered ordering.

## Reliance on deprecated routing behavior {#deprecated}
**Looks like.** Code depending on `$app/stores` for `page`/`navigating`, or on
routing behavior removed or changed in the installed major.
**Detect.** Imports from deprecated modules, and advice or code that assumes an
older SvelteKit's conventions.
**Why it hurts.** Deprecated APIs are removed in future majors, so the routing
layer carries upgrade debt and may break on the next update.
**Fix.** Migrate to the current stable equivalents (for example `$app/state` in
place of `$app/stores`, which requires runes), and re-verify any behavior against
the docs for the installed version. See `references/version-verification.md`.
