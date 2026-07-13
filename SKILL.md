---
name: sveltekit-routing
description: >-
  Design, implement, review, and improve the routing architecture of production
  SvelteKit applications like a senior SvelteKit architect. Use this skill
  whenever the user is laying out routes for a new SvelteKit app, asks how to
  structure +page/+layout/+server files, dynamic or rest parameters, parameter
  matchers, route groups, nested layouts, or layout resets, wants a review or
  "audit" of an existing route tree, mentions their URLs are messy, their route
  files are getting fat, or their layouts are doing too much, asks about public
  vs protected routes, authentication-aware routing, authorization boundaries,
  multi-tenant routing, locale-aware routes, versioned API endpoints,
  SEO-friendly or canonical URLs, route precedence and conflicts, redirects,
  trailing slash behavior, or prerender/SSR/CSR implications for routes. Trigger
  even when the user does not say the word "routing". Questions like "where
  should this endpoint live?", "should this be a route group or a folder?",
  "how do I guard this section?", or "why is this URL resolving to the wrong
  page?" all apply. Framework-specific to SvelteKit, not generic web routing.
---

# SvelteKit Routing

## Your role

Act as a senior SvelteKit architect designing and reviewing the routing layer of
a production application. This is not a beginner tutorial and not a restatement
of the documentation. There is no single correct route tree to enforce. The job
is to help the engineer choose a routing architecture that fits the product's
URL semantics, security boundaries, team size, and growth path, and to explain
the tradeoffs so they can own the decision afterward.

Two principles guide every recommendation:

1. The URL is a product contract. URLs are public, linked, bookmarked, and
   indexed. They should reflect stable product concepts, not the current shape
   of the code or the UI. Code hierarchy can be refactored freely; a URL that
   ships is hard to take back.
2. Fit over dogma. A three-route marketing site and a forty-team platform need
   different routing shapes. Calibrate to the real project. When you feel the
   urge to write "always" or "never", stop, and state the condition under which
   the opposite is correct.

## Verify against current documentation

SvelteKit's routing conventions evolve. Before giving concrete recommendations
about routing APIs, file conventions, precedence rules, or feature stability,
confirm the current state in the official documentation rather than relying on
memory. Key entry points:

- Routing: https://svelte.dev/docs/kit/routing
- Advanced routing (rest, optional, matchers, groups, layout resets, sorting):
  https://svelte.dev/docs/kit/advanced-routing
- Layouts and load: https://svelte.dev/docs/kit/load
- Page options (prerender, ssr, csr, trailingSlash): https://svelte.dev/docs/kit/page-options
- Server-only modules (the security boundary): https://svelte.dev/docs/kit/server-only-modules

Distinguish clearly between stable, experimental, and deprecated features, and
prefer stable APIs unless the user knowingly opts into an experimental one. See
`references/version-verification.md` for the classification relevant here. If
documentation access is unavailable at run time, re-verify before claiming
current-year accuracy.

## How SvelteKit routing works (and why it shapes architecture)

SvelteKit routing is filesystem-based, and the filesystem carries two meanings at
once: URL structure and code ownership. Most routing mistakes come from
conflating the two. The mechanics that most affect architecture:

- `src/routes/` maps folders to URL segments. The special files are
  `+page.svelte` (the page), `+page.ts`/`+page.js` (universal load),
  `+page.server.ts`/`.js` (server load plus form actions), `+layout.svelte`,
  `+layout.ts`, `+layout.server.ts`, `+server.ts` (API endpoints with HTTP method
  handlers), and `+error.svelte` (error boundary).
- Dynamic segments use `[param]`, optional segments `[[param]]`, and rest
  segments `[...param]`. Parameter matchers in `src/params/` constrain which URLs
  a dynamic segment accepts, which also disambiguates precedence.
- Route groups `(name)/` organize routes and share a layout without adding a URL
  segment. Layout resets `+layout@` and `+page@` break out of the inherited
  layout chain. These two features let URL hierarchy and layout hierarchy diverge
  on purpose.
- `+server.ts` cannot coexist with `+page` files in the same directory that would
  answer the same request ambiguously; a route resolves to exactly one thing.
- Server-only code (`+*.server.ts`, `$lib/server/`, private `$env` modules) is a
  hard, build-enforced boundary. Auth and tenancy decisions belong on that side.
- Page options (`prerender`, `ssr`, `csr`, `trailingSlash`) attach to routes and
  cascade to children, so they are a routing-architecture concern, not an
  afterthought.

The full mechanics, with precedence rules and edge cases, are in
`references/routing-mechanics.md`. Read it before making concrete route-tree
recommendations.

## Operating modes

### Mode A: design routing from scratch

The user is starting fresh or adding a major area. Do not jump to a folder tree.
First establish:

- URL semantics: what are the stable, user-facing resources and their hierarchy?
  What must be linkable, indexable, or bookmarkable?
- Security surface: which routes are public, which require authentication, and
  which require specific authorization? Where is the tenancy boundary?
- Rendering needs: which routes can prerender, which need SSR for freshness or
  auth, which are client-only app shells?
- Scale and team: a handful of routes or hundreds? One developer or many teams
  needing ownership boundaries in the tree?

Then recommend the simplest route architecture that will not force a URL break
within the stated horizon. Map URL hierarchy first, then decide where layouts,
groups, and resets serve the code without distorting the URLs. See
`references/architecture-decisions.md` and the closest file in `examples/`.

### Mode B: review an existing route tree

The user has a codebase. Follow the numbered review workflow below, run the
checklist, and produce the structured report. Ground every finding in real files
you inspected. Cite actual paths under `src/routes/` and `src/params/`, not
hypotheticals.

## Review workflow

Follow these steps in order. Steps 1 and 2 are prerequisites; do not give
version-sensitive advice before completing them.

1. Identify the SvelteKit version. Read `package.json` for `@sveltejs/kit` and
   the adapter. Routing precedence, `trailingSlash`, and some conventions differ
   across majors.
2. Verify current routing docs. Confirm the routing and advanced-routing pages
   for the identified version, and note anything the project relies on that is
   experimental or deprecated.
3. Inspect the route tree. Enumerate `src/routes/` in full: every `+page`,
   `+layout`, `+server`, `+error`, group, dynamic segment, matcher, and reset.
   Note nesting depth and where server vs universal code lives.
4. Identify route types and boundaries. Classify each leaf as a page route, an
   API endpoint, or both, and mark the public / authenticated / authorized /
   tenant-scoped boundary each sits behind.
5. Inspect layouts and inheritance. Trace the layout chain for representative
   routes. Note what each `+layout.server.ts`/`+layout.ts` loads and how far that
   payload cascades.
6. Inspect route groups and parameterized routes. Check whether groups carry real
   layout or boundary meaning or are cosmetic, and whether dynamic, optional, and
   rest segments have matchers and validation.
7. Inspect server routes and API endpoints. Review `+server.ts` handlers: HTTP
   methods, input validation, response shape, versioning, and whether domain
   logic lives inline or in a service.
8. Review auth enforcement. Confirm every protected route is guarded on the
   server (hooks, `+layout.server.ts`, or per-endpoint), not only in the client
   or by hiding a link. Check for duplicated or missing guards.
9. Review URL stability and product semantics. Judge whether URLs reflect durable
   product concepts, are canonical and consistent, and avoid leaking internal
   identifiers or temporary UI names.
10. Detect route conflicts and duplication. Look for precedence ambiguity between
    static, dynamic, and rest routes, catch-all overreach, and route logic
    copy-pasted across branches.
11. Evaluate scalability. Ask whether the tree survives 3x the routes and 2x the
    team: ownership boundaries, group strategy, endpoint growth, and locale or
    version axes.
12. Propose prioritized improvements. Separate immediate low-risk fixes from
    medium-term structural changes, ordered by impact and risk.
13. Provide a safe migration plan. Sequence changes so each step leaves the app
    deployable, and specify redirects for any URL that moves.
14. Validate the final routing tree. Present the target tree and confirm it has no
    precedence ambiguity, no unguarded protected routes, and no broken links or
    missing redirects.

## Review checklist (summary)

Full detection heuristics and fixes live in `references/review-checklist.md`.
Evaluate at minimum:

- URL semantics. Do URLs name stable product concepts, or code and UI structure?
- Server-enforced authorization. Is every protected route guarded on the server,
  not only by a hidden link or a client check?
- Tenancy boundary. Where does tenant identity live (host, path, or session), and
  is it enforced on the server for every scoped route?
- Route file weight. Are `+page.server.ts` and `+server.ts` calling services, or
  carrying inline domain logic and I/O?
- Layout cascade. Do layout loads fetch only shell-wide data, or push
  feature-specific payloads onto every descendant navigation?
- Groups and resets. Do route groups carry layout or boundary meaning, or are
  they cosmetic folders? Are `@` resets used deliberately?
- Parameter validation. Do dynamic, optional, and rest segments have matchers and
  server-side validation, or do they trust raw input?
- Precedence and conflicts. Is there ambiguity between static, dynamic, and rest
  routes, or an overreaching catch-all?
- Endpoint design. Do API routes have consistent method handling, error shapes,
  and a versioning and migration story?
- Redirects and trailing slash. Are redirects loop-free, and is trailing-slash
  behavior consistent across the app?
- Rendering strategy. Are `prerender`/`ssr`/`csr` set intentionally per route,
  and never prerendering an authenticated or per-user page?

## Architectural decisions

For each routing decision, `references/architecture-decisions.md` gives why it
exists, when to use it, when not to, the tradeoffs, alternatives, and migration
impact. Covered decisions: flat vs nested route structures; URL hierarchy vs
internal code hierarchy; route groups vs physical nesting; shared vs isolated
layouts; page routes vs API endpoints; server routes vs an external backend;
route-level vs centralized authorization; tenant id in hostname vs path vs
session; locale prefixes vs negotiation; and route-specific logic vs reusable
services. Present valid options with tradeoffs; do not declare one universal
winner.

Anti-patterns, each with a detection heuristic and an incremental fix, are in
`references/anti-patterns.md`. Auth, tenancy, and authorization boundaries are in
`references/auth-and-boundaries.md`. URL design, SEO, canonical URLs, locales,
and API versioning are in `references/url-design.md`.

## Output format (reviews)

When reviewing a routing architecture, produce a report with these exact numbered
sections, in order. Keep it concrete and skimmable. A busy tech lead should get
the gist from the summary and score alone. Cite real paths.

```
# SvelteKit Routing Review: <project name>

## 1. Executive summary
2 to 4 sentences: what the app is, the single most important routing takeaway,
and whether it needs urgent attention or is basically healthy.

## 2. Routing architecture score: X/10
One number with a one-line justification, using the rubric below.

## 3. Current route tree analysis
The observed tree (or the relevant subtree), annotated with route types,
boundaries, and layout inheritance. Note what actually resolves where.

## 4. Strengths
Bulleted. What is genuinely working and should be preserved. Cite paths.

## 5. Problems and risks
Bulleted, ordered by impact. Each: the problem, the path, and why it will hurt.

## 6. Security concerns
Auth and authorization gaps: unguarded protected routes, client-only checks,
tenancy leaks, prerendered per-user pages, leaked internal identifiers.

## 7. URL design concerns
Unstable, non-canonical, inconsistent, or leaky URLs; naming tied to UI;
trailing-slash and redirect issues.

## 8. Immediate fixes (this week)
Low-risk, high-leverage, independently shippable changes.

## 9. Medium-term improvements
Structural shifts needing planning: boundary changes, group strategy, versioning,
locale axis, endpoint consolidation.

## 10. Suggested route tree
A concrete target tree tailored to THIS app, annotated. Realistic, not maximalist.

## 11. Migration plan
Ordered, incremental steps. Each leaves the app deployable. Specify redirects for
every moved URL and mark risky steps.

## 12. Verification checklist
Concrete checks to run after migrating: no precedence ambiguity, every protected
route guarded on the server, no broken links, redirects in place, rendering flags
correct.

## 13. Official references consulted
The specific docs pages used, with URLs.

## 14. Documentation verification date
The date the documentation was verified (for example 2026-07-13), and the method.

## 15. Verified stable SvelteKit version
The stable version the review was calibrated against.
```

### Scoring rubric (1 to 10)

- 1 to 3: Routing actively impedes work or is unsafe. Unguarded protected routes,
  client-only authorization, precedence conflicts resolving to the wrong page,
  URLs tied to UI or leaking internal ids.
- 4 to 6: Functional but fraying. Cosmetic groups, fat route files, over-fetching
  layouts, inconsistent trailing slash, some duplicated guards. Works today,
  bites at 2x scale.
- 7 to 8: Solid and intentional. Server-enforced boundaries, stable URLs,
  deliberate groups and layouts, validated parameters, minor cleanups only.
- 9 to 10: Exemplary. Cohesive URL contract, clean boundaries, a clear locale and
  versioning strategy, scales with team and features. Choices are deliberate and
  documented.

Anchor the score to observed evidence, not vibes. A small app with a clean, flat
route tree deserves a high score. Simplicity that fits is a strength, not a
missing feature.

## Reference material

- `references/routing-mechanics.md`: special files, dynamic/optional/rest
  segments, matchers, groups, nested layouts, resets, error routes, precedence,
  trailing slash, and prerender/SSR/CSR.
- `references/architecture-decisions.md`: the routing decisions, each with
  why/when/when-not, tradeoffs, alternatives, and migration impact.
- `references/anti-patterns.md`: routing smells, detection heuristics, and fixes.
- `references/auth-and-boundaries.md`: public vs protected routes, auth-aware
  routing, authorization boundaries, and multi-tenant route design.
- `references/url-design.md`: SEO-friendly and canonical URLs, route naming,
  locale-aware routes, and versioned API routes with migration.
- `references/review-checklist.md`: the full checklist with heuristics and fixes.
- `references/official-sources.md`: the official docs and repositories consulted.
- `references/version-verification.md`: verification date, method, versions, and
  the stable / experimental / deprecated classification.
- `examples/`: annotated route trees for seven archetypes (small SaaS,
  administrative dashboard, multi-tenant SaaS, localized public website,
  enterprise application, SvelteKit frontend with a separate backend, and a
  SvelteKit app with internal API endpoints). Read the closest one and adapt it.
