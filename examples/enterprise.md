# Example: Enterprise Application

**Profile.** Many teams, hundreds of routes, several product areas, strict access
control, long lifespan. Ownership boundaries in the tree matter as much as URLs.

**Routing approach.** Product areas as top-level segments that map to team
ownership. A shared authed shell, per-area layouts and guards, matcher-validated
ids, and a versioned public API for external integrators. Route groups separate
concerns that must not appear in the URL; real segments carry meaningful prefixes.

## Route tree

```
src/routes/
├── (public)/                      # marketing, status, docs
├── (auth)/login/+page.server.ts
├── (app)/
│   ├── +layout.server.ts          # session + org context (shell-wide only)
│   ├── +layout.svelte
│   ├── billing/                    # owned by Billing team
│   │   ├── +layout.server.ts       # requirePermission('billing:*')
│   │   ├── +page.svelte
│   │   └── invoices/[id=uuid]/+page.server.ts
│   ├── projects/                   # owned by Projects team
│   │   ├── +layout.server.ts
│   │   └── [id=uuid]/settings/+page.server.ts
│   └── admin/                      # owned by Platform team
│       ├── +layout.server.ts       # requireRole('admin')
│       └── users/[id=uuid]/+page.server.ts
├── api/
│   ├── v1/…/+server.ts             # external integrators; versioned
│   └── v2/…/+server.ts             # shares services with v1
└── src/params/uuid.ts
```

## Key code

Coarse gate at the shell, area authorization at each area layout:

```ts
// src/routes/(app)/+layout.server.ts  (authentication + org, shell-wide)
export const load = async ({ locals, url }) => {
  if (!locals.user) throw redirect(303, `/login?redirectTo=${url.pathname}`);
  return { user: locals.user, org: locals.org };
};

// src/routes/(app)/billing/+layout.server.ts  (area authorization)
import { requirePermission } from '$lib/server/authz';
export const load = async ({ locals }) => {
  requirePermission(locals, 'billing:read');
  return {};
};
```

Two API versions, one implementation:

```ts
// src/routes/api/v1/invoices/+server.ts
import { listInvoices } from '$lib/server/billing/service';
import { toV1 } from './serialize';
export const GET = async ({ locals }) =>
  json((await listInvoices(locals.org.id)).map(toV1)); // v2 differs only in shape
```

## Why this shape
- Top-level product segments (`billing`, `projects`, `admin`) map to team
  ownership, so a team owns a subtree end to end and boundaries are legible.
- Authentication is enforced once at the `(app)` shell; each area adds its own
  authorization layout, so a new route in an area inherits both.
- The public API is versioned by path for external integrators, but both versions
  call the same `$lib/server/` services; only serialization differs, so the domain
  logic is not forked.
- Matchers on ids keep malformed URLs out of load, and ownership is still checked
  in each route.

## Scaling notes
- Reach for route groups only where a shared layout or guard justifies them; do not
  add cosmetic groups to tidy a large tree.
- When a locale or a second version axis appears, plan it deliberately (see the
  localized and API-endpoint examples) rather than retrofitting under pressure.

## Score anchor
8 to 10 when boundaries are server-enforced, ownership maps to the tree, and API
versions share services. Dock for duplicated guards, cosmetic groups, or forked
version logic.

## References
- Advanced routing: https://svelte.dev/docs/kit/advanced-routing
- Hooks: https://svelte.dev/docs/kit/hooks
