# Example: Administrative Dashboard

**Profile.** An internal or admin dashboard behind login. SEO is irrelevant, most
pages are per-user and data-heavy, and some views need to escape the app chrome.

**Routing approach.** One guarded shell. Role-based authorization per section.
Client-rendered internals where SEO does not matter. Layout resets for full-screen
views. A small internal API for widgets that poll.

## Route tree

```
src/routes/
├── (auth)/login/+page.server.ts
├── (dash)/
│   ├── +layout.server.ts        # require session; load user + role + nav
│   ├── +layout.svelte           # sidebar + topbar shell
│   ├── +page.svelte             # /  overview
│   ├── users/
│   │   ├── +page.server.ts      # requirePermission(locals, 'users:read')
│   │   └── [id=uuid]/+page.server.ts   # ownership/role check + load
│   ├── reports/
│   │   ├── +page.svelte
│   │   └── [id=uuid]/
│   │       ├── +page.svelte
│   │       └── +page@(dash).svelte  # reset: full-screen report, no sidebar
│   └── settings/+page.server.ts
├── api/
│   └── metrics/+server.ts        # GET, polled by widgets; guarded like pages
└── src/params/uuid.ts            # matcher: reject non-uuid ids
```

## Key code

Role gate as a shared helper, applied per section:

```ts
// src/lib/server/authz.ts
import { error } from '@sveltejs/kit';
export function requirePermission(locals: App.Locals, perm: string) {
  if (!locals.user) throw error(401);
  if (!locals.user.permissions.includes(perm)) throw error(403);
}

// src/routes/(dash)/users/+page.server.ts
import { requirePermission } from '$lib/server/authz';
export const load = async ({ locals }) => {
  requirePermission(locals, 'users:read');
  return { users: await listUsers(locals.tenantId) };
};
```

Client-only rendering for a heavy internal-only view:

```ts
// src/routes/(dash)/reports/+page.ts
export const ssr = false; // internal, no SEO; ship a client shell
```

## Why this shape
- One `(dash)` group carries the session guard and the shell; every child inherits
  the boundary.
- Authorization is per section via a shared helper, not duplicated inline, and the
  polled `/api/metrics` endpoint reuses the same guard so it is not a hole.
- `+page@(dash).svelte` resets the layout for a full-screen report without moving
  its URL out of `/reports/[id]`.
- The `uuid` matcher turns malformed ids into 404s before load runs; existence and
  authorization are still checked in load.

## Score anchor
7 to 9 out of 10 when boundaries are server-enforced and resets are deliberate.
Dock for client-only guards or inline duplicated role checks.

## References
- Advanced routing (resets, matchers): https://svelte.dev/docs/kit/advanced-routing
- Page options (`ssr`): https://svelte.dev/docs/kit/page-options
