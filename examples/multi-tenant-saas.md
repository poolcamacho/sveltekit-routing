# Example: Multi-tenant SaaS

**Profile.** Many customer organizations (tenants) share one deployment. Each user
belongs to one or more tenants. Cross-tenant data leakage is the top risk.

**Routing approach.** Path-based tenancy (`/t/[tenant]/...`) for one-domain
simplicity and shareable URLs, with the tenant boundary enforced on the server for
every request. The tenant segment is a hint; the server-side membership check and
query scoping are the boundary.

Two other placements exist (hostname per tenant, session-scoped tenant); see
`references/architecture-decisions.md` for the tradeoffs. Path-based is shown here
because it deploys on one domain and keeps URLs shareable.

## Route tree

```
src/routes/
├── (public)/
│   ├── +page.svelte                    # marketing
│   └── login/+page.server.ts
├── (app)/
│   └── t/
│       └── [tenant=slug]/              # tenant slug, matcher-validated shape
│           ├── +layout.server.ts       # verify membership; scope everything
│           ├── +layout.svelte          # tenant-branded shell
│           ├── +page.svelte            # /t/acme
│           ├── projects/
│           │   ├── +page.server.ts
│           │   └── [id=uuid]/+page.server.ts
│           └── settings/+page.server.ts
└── src/params/slug.ts                  # matcher: tenant-slug shape only
```

## Key code

Resolve identity in hooks, enforce membership and scope in the tenant layout:

```ts
// src/hooks.server.ts
export const handle = async ({ event, resolve }) => {
  event.locals.user = await resolveSession(event.cookies);
  return resolve(event);
};

// src/routes/(app)/t/[tenant]/+layout.server.ts
import { error, redirect } from '@sveltejs/kit';
export const load = async ({ params, locals, url }) => {
  if (!locals.user) throw redirect(303, `/login?redirectTo=${url.pathname}`);
  const membership = await findMembership(locals.user.id, params.tenant);
  if (!membership) throw error(404); // do not confirm the tenant exists
  locals.tenantId = membership.tenantId;   // internal id, never the URL slug
  return { tenant: membership.tenant, role: membership.role };
};
```

Every query scopes by the resolved internal tenant id, not the URL slug:

```ts
// src/routes/(app)/t/[tenant]/projects/+page.server.ts
export const load = async ({ locals }) => {
  return { projects: await listProjects(locals.tenantId) }; // scoped server-side
};
```

## Why this shape
- The tenant slug is validated for shape by a matcher, but membership is checked in
  server code; editing the URL to another tenant yields a 404, not data.
- Queries use the internal `tenantId` resolved from membership, never the raw URL
  slug, so a mis-routed request cannot read another tenant.
- Public routes sit outside `(app)` so login and marketing are reachable without a
  tenant.
- URLs are shareable within a tenant and the deployment stays on one domain.

## Migration note
Changing tenant placement later (path to host, or path to session) rewrites every
tenant-scoped URL and the auth flow. Decide early. If unsure, path-based leaves the
most options open.

## Score anchor
8 to 10 when membership is enforced server-side and every query is tenant-scoped.
Any trust in the raw tenant segment caps the score in the 1 to 3 band; it is a
data-isolation failure.

## References
- Advanced routing (matchers): https://svelte.dev/docs/kit/advanced-routing
- Hooks: https://svelte.dev/docs/kit/hooks
- Server-only modules: https://svelte.dev/docs/kit/server-only-modules
