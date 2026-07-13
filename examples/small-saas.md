# Example: Small SaaS

**Profile.** 1 to 3 engineers, roughly 10 to 20 routes, a few areas (marketing,
auth, an authed app, a webhook). Shipping fast, URL shape still settling.

**Routing approach.** Three route groups separate audiences without adding URL
segments. One server guard in the authed group. Endpoints only where a machine
client needs one. Do not build a locale or version axis yet.

## Route tree

```
src/routes/
├── (marketing)/                 # public, indexable, no guard
│   ├── +layout.svelte           # marketing shell
│   ├── +page.svelte             # /
│   └── pricing/+page.svelte     # /pricing
├── (auth)/                      # public, kept OUTSIDE the guarded group
│   ├── login/+page.server.ts    # form action -> session
│   └── signup/+page.server.ts
├── (app)/                       # authenticated area, guarded once here
│   ├── +layout.server.ts        # session guard + nav data (shell-wide)
│   ├── +layout.svelte
│   ├── dashboard/+page.svelte   # /dashboard
│   └── settings/
│       ├── +page.svelte         # /settings
│       └── billing/+page.server.ts   # /settings/billing (permission check)
└── api/
    └── webhooks/stripe/+server.ts    # machine client: POST only
```

## Key code

Central identity in hooks, coarse guard in the group layout:

```ts
// src/hooks.server.ts
export const handle = async ({ event, resolve }) => {
  event.locals.user = await resolveSession(event.cookies);
  return resolve(event);
};

// src/routes/(app)/+layout.server.ts
import { redirect } from '@sveltejs/kit';
export const load = async ({ locals, url }) => {
  if (!locals.user) throw redirect(303, `/login?redirectTo=${url.pathname}`);
  return { user: locals.user, nav: navFor(locals.user) };
};
```

## Why this shape
- The `(auth)` group stays public so the guard cannot lock users out of login.
- Authentication is enforced on the server, once, for the whole `(app)` subtree;
  a new route under `(app)` inherits it.
- The webhook is a `+server.ts` because the caller is a machine; in-app mutations
  use form actions instead.
- No `[[lang]]` prefix, no `/api/v1`: adding them now would be ceremony. Leave room
  by keeping URLs clean.

## Score anchor
A clean version of this deserves 7 to 8 out of 10. Simplicity that fits is a
strength; do not dock it for lacking locale or versioning it does not need.

## References
- Advanced routing (groups): https://svelte.dev/docs/kit/advanced-routing
- Hooks: https://svelte.dev/docs/kit/hooks
