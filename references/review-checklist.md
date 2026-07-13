# Routing Review Checklist

Run this during a Mode B review. Each item has a detection heuristic and a
remediation. Ground every finding in a real path under `src/routes/` or
`src/params/`. This checklist feeds the numbered output format in SKILL.md.

## 1. URL semantics and stability
- **Detect.** Route folders named after UI, experiments, frameworks, or code
  structure; frequent redirect churn in git history; internal ids in params.
- **Fix.** Rename to durable product concepts, move code grouping to `$lib` and
  route groups, use opaque ids, and treat URL changes as migrations with
  redirects. See `references/url-design.md`.

## 2. Server-enforced authentication
- **Detect.** Protected routes guarded only in `.svelte` or universal load;
  request a protected URL directly while logged out and see if it serves.
- **Fix.** Enforce in `hooks.server.ts` or the section `+layout.server.ts`; keep
  login routes outside the guarded group. See `references/auth-and-boundaries.md`.

## 3. Route-specific authorization
- **Detect.** Authenticated-but-not-authorized access: any logged-in user can open
  another user's resource or an admin route; ownership not checked in load.
- **Fix.** Check ownership and permissions server-side per resource route, via a
  shared `requirePermission` helper; prefer `error(404)` for unauthorized reads.

## 4. Tenancy boundary
- **Detect.** Tenant param or host trusted without a membership check; queries not
  scoped by tenant; editing the tenant segment reaches another tenant's data.
- **Fix.** Resolve tenant in hooks, verify membership in the section layout, and
  scope every query by tenant on the server. See `references/auth-and-boundaries.md`.

## 5. Route file weight and logic placement
- **Detect.** `+page.server.ts`/`+server.ts` far larger than siblings; domain
  logic and inline queries in load/actions; no `$lib` import.
- **Fix.** Extract domain logic and I/O into `$lib/server/` services; keep routes
  to validate, call, shape. See `references/anti-patterns.md`.

## 6. Layout cascade
- **Detect.** Layout loads returning feature-specific or large payloads; heavy
  queries re-running on every child navigation.
- **Fix.** Restrict layout loads to shell-wide data; push feature data into the
  specific page load or a nested feature layout.

## 7. Route groups and layout resets
- **Detect.** Groups with no shared layout, guard, or error boundary; constant use
  of `@` resets; a group that only tidies the explorer.
- **Fix.** Remove cosmetic groups or promote to real segments; use resets
  deliberately for routes that must escape the section chrome.

## 8. Parameterized routes and matchers
- **Detect.** Dynamic, optional, or rest segments with no matcher and no runtime
  validation; params passed straight into queries, `fetch`, or the filesystem.
- **Fix.** Add matchers for shape, validate existence and authorization in load,
  and constrain rest params before any filesystem use. See
  `references/routing-mechanics.md`.

## 9. Server routes and endpoints
- **Detect.** `+server.ts` mixing transport and domain logic; other files importing
  from a `+` file; inconsistent method handling; no shared error shape.
- **Fix.** Thin handlers calling shared services; explicit method handlers; nothing
  imports from route files.

## 10. Precedence and conflicts
- **Detect.** Routes that resolve to the wrong page; behavior that changed after a
  sibling was added; catch-alls placed high in the tree.
- **Fix.** Add matchers to disambiguate static vs dynamic vs rest; scope catch-alls
  narrowly; confirm sorting against the advanced-routing docs.

## 11. Redirects and trailing slash
- **Detect.** "Too many redirects"; guards that redirect without an idempotency
  check; mixed trailing-slash forms; canonical tags disagreeing with redirects.
- **Fix.** Make guards idempotent; set one `trailingSlash` policy high in the tree;
  align canonical tags and redirects; verify the adapter/CDN rules.

## 12. Rendering strategy per route
- **Detect.** `prerender = true` on an authenticated or per-user page; `ssr =
  false` on indexable content; rendering flags set by habit, not intent.
- **Fix.** Prerender only shared, non-per-request pages; SSR anything auth-
  dependent or indexable; set flags deliberately, remembering they cascade.

## 13. Duplication across branches
- **Detect.** Near-identical load/guard bodies across groups or sections; the same
  data-shaping copied in several routes.
- **Fix.** Extract shared load logic into `$lib`; use a subtree layout load where
  the data is genuinely shared; extract only true sameness.

## 14. Endpoint versioning and migration
- **Detect.** Multiple `/api/vN` prefixes with no sunset plan, or an unversioned
  public API that shipped breaking changes.
- **Fix.** Version at a deliberate boundary for external consumers, share the
  service implementation, publish a deprecation and sunset policy with telemetry.

## 15. Scalability and ownership
- **Detect.** No ownership boundaries in a large tree; a group strategy that will
  not survive adding locales or versions; endpoints multiplying without structure.
- **Fix.** Introduce group and segment boundaries that map to teams and product
  areas; plan the locale and version axes before they are forced.

## 16. Deprecated behavior
- **Detect.** Imports from `$app/stores` for `page`/`navigating`; reliance on
  routing behavior changed in the installed major.
- **Fix.** Migrate to current stable equivalents (`$app/state`, which requires
  runes) and re-verify against the docs for the installed version. See
  `references/version-verification.md`.
