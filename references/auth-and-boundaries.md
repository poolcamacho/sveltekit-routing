# Authentication, Authorization, and Tenancy Boundaries

Routing is where access control is enforced or lost. This file covers public vs
protected routes, authentication-aware routing, authorization boundaries, and
multi-tenant route design. The governing rule: the URL and the client are never
the boundary; server code is. Verify server-only-module behavior at
https://svelte.dev/docs/kit/server-only-modules and hooks at
https://svelte.dev/docs/kit/hooks.

## Table of contents
- [The boundary is on the server](#server-boundary)
- [Public vs protected routes](#public-vs-protected)
- [Where to enforce: hooks, layout, route](#where)
- [Authentication-aware routing patterns](#auth-aware)
- [Authorization boundaries](#authz)
- [Multi-tenant route design](#tenancy)
- [Common failures](#failures)

## The boundary is on the server {#server-boundary}

Three facts drive every decision here:

1. `+page.svelte` and universal `+page.ts`/`+layout.ts` run in the browser on
   client navigation. Nothing checked only there is trusted.
2. `+*.server.ts`, `$lib/server/`, and `hooks.server.ts` run only on the server.
   This is the boundary, enforced by the build.
3. Hiding a link or rendering `{#if user}` is UX, not access control. The URL and
   the endpoint must reject unauthorized requests on their own.

Design the route tree so that every protected URL passes through server code that
can say no, and so that a newly added route inherits that check by default rather
than needing the author to remember it.

## Public vs protected routes {#public-vs-protected}

Separate the two at the tree level so the boundary is structural, not per-file.

- Put public routes (marketing, login, signup, public content) in one area, often
  a `(public)` or `(marketing)` group, with no guard.
- Put authenticated routes in an `(app)` group (or an `/app` segment) whose
  `+layout.server.ts` establishes the session and rejects anonymous requests.
- Keep the login and signup routes outside the guarded group so the guard cannot
  lock users out of the page that lets them in (a common redirect-loop cause).

This makes "is this route protected?" answerable from the tree, and makes it hard
to add an unguarded route inside the protected area by accident.

## Where to enforce: hooks, layout, route {#where}

Three enforcement points, used together:

- `hooks.server.ts` (`handle`): resolve the session once per request and attach
  the user and tenant to `event.locals`. This is the single source of identity.
  It can also block whole path prefixes early.
- Section `+layout.server.ts`: read `locals` and reject anonymous or wrong-tenant
  requests for the whole subtree, and return shell data (user, nav). This is the
  coarse gate for authentication and tenancy.
- Route `+page.server.ts` / `+server.ts`: check the specific permission this route
  needs (`billing:admin`, resource ownership) and validate the resource exists and
  belongs to the caller.

Guidance: authenticate and resolve tenancy centrally (hooks plus section layout);
authorize specific permissions per route. Do not scatter authentication checks
across every route (they drift and get forgotten), and do not push fine-grained
permission logic into a hook where it becomes a tangled dispatch.

## Authentication-aware routing patterns {#auth-aware}

- Redirect anonymous users from protected routes to login, preserving the intended
  destination (for example `redirect(303, `/login?redirectTo=${url.pathname}`)`),
  and validate `redirectTo` is a local path before using it so it cannot become an
  open redirect.
- After login, send the user to the validated `redirectTo` or a safe default.
  Ensure the login route itself does not redirect authenticated users in a way
  that loops.
- For routes that differ by auth state (a landing page vs the app), branch in
  server load and return different data or redirect, rather than rendering both
  and hiding one client-side.
- Do not prerender authenticated routes. A prerendered page is identical for
  everyone; an auth-dependent page must be server-rendered per request. See
  `references/routing-mechanics.md`.

## Authorization boundaries {#authz}

Authentication answers "who are you"; authorization answers "may you do this".
Keep them distinct in the tree:

- Model permissions as data (roles, scopes) resolved server-side, not as route
  names. A route named `/admin` is a convenience, not a guarantee; the guard is
  what enforces it.
- Check ownership on every resource route: the fact that a user is authenticated
  does not mean this order, tenant, or document is theirs. Validate in server load
  and throw `error(403)` or `error(404)` (prefer 404 to avoid confirming existence
  to an unauthorized caller).
- Centralize the permission helper (`requirePermission(locals, 'x')`) so routes
  call one implementation. Duplicated inline checks drift and leave gaps.
- Apply the same authorization to the page and to any endpoint that exposes the
  same data. A guarded page with an unguarded `+server.ts` sibling is a hole.

## Multi-tenant route design {#tenancy}

Tenancy is an authorization boundary expressed partly through routing. Decide
where the tenant lives (see `references/architecture-decisions.md` for the full
tradeoff), then enforce it on the server for every scoped request.

- Hostname (`acme.app.com`): resolve the tenant from `event.url.host` in
  `hooks.server.ts`, attach to `locals.tenant`, and confirm the user belongs to
  it. Needs wildcard DNS/TLS and adapter support.
- Path (`/t/[tenant]/...`): the tenant is a route param. Add a matcher for its
  shape, and in the section `+layout.server.ts` verify the authenticated user is a
  member of `params.tenant`. Never trust the path segment alone; a user can edit
  the URL to another tenant.
- Session (tenant chosen after login): read `locals.tenant` from the session; URLs
  stay tenant-free but are not shareable across tenants.
- In all cases: scope every query by tenant in server code, so even a mis-routed
  or hostile request cannot read another tenant's data. The tenant in the URL is a
  hint; the server-side scope is the boundary.

## Common failures {#failures}

- Guard only in `+layout.svelte` or universal load: bypassable by direct request.
  Move it to `+*.server.ts` or hooks.
- Guarded page, unguarded endpoint: the `+server.ts` that serves the same data has
  no check. Guard both, ideally via a shared helper.
- Tenant param trusted without membership check: user edits the URL to another
  tenant and reads their data. Verify membership and scope queries server-side.
- Login route inside the guarded group: anonymous users are redirected away from
  the page that would authenticate them. Keep login public.
- Redirect loop between guard and login: make the guard idempotent and validate
  `redirectTo`.
- Prerendered per-user route: one user's page shipped to all. Never prerender
  authenticated routes.
- Enumerable ids in guarded routes: even with a guard, sequential ids leak counts
  and invite probing. Use opaque ids and still authorize the resource.
