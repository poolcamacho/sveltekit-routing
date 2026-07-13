# Example: SvelteKit Frontend with a Separate Backend API

**Profile.** The domain lives in an existing external API (another team, another
repo, or a polyglot service). SvelteKit is the frontend and a thin
backend-for-frontend (BFF). Secrets and tokens must stay server-side.

**Routing approach.** No domain `+server.ts` endpoints that reimplement the API.
Server load functions call the external API, keeping the token on the server. A
minimal BFF endpoint exists only where the browser must call something directly
(for example a token-scoped upload) and even then it proxies through the server.
See `references/architecture-decisions.md` for server-routes-vs-external-backend.

## Route tree

```
src/routes/
├── (auth)/login/+page.server.ts     # exchanges credentials -> session cookie
├── (app)/
│   ├── +layout.server.ts            # session guard; expose user, not the token
│   ├── +layout.svelte
│   ├── orders/
│   │   ├── +page.server.ts          # server load -> external API (token server-side)
│   │   └── [id]/+page.server.ts
│   └── profile/+page.server.ts
└── bff/
    └── search/+server.ts            # thin proxy for client-driven typeahead
```

## Key code

A single server-side API client that attaches the secret token:

```ts
// src/lib/server/api.ts   (server-only: token never reaches the client)
import { API_BASE, API_TOKEN } from '$env/static/private';
export async function apiGet(path: string, fetchFn: typeof fetch) {
  const res = await fetchFn(`${API_BASE}${path}`, {
    headers: { authorization: `Bearer ${API_TOKEN}` }
  });
  if (!res.ok) throw error(res.status);
  return res.json();
}

// src/routes/(app)/orders/+page.server.ts
import { apiGet } from '$lib/server/api';
export const load = async ({ fetch, locals }) => {
  return { orders: await apiGet(`/orders?user=${locals.user.id}`, fetch) };
};
```

A thin BFF endpoint for client-initiated calls, still server-guarded:

```ts
// src/routes/bff/search/+server.ts
import { apiGet } from '$lib/server/api';
export const GET = async ({ url, fetch, locals }) => {
  if (!locals.user) throw error(401);
  const q = url.searchParams.get('q') ?? '';
  return json(await apiGet(`/search?q=${encodeURIComponent(q)}`, fetch));
};
```

## Why this shape
- The external API token lives in `$env/static/private` and is used only in
  `$lib/server/`, so it cannot reach the browser bundle.
- Load functions are the single data-access seam. If the backend later moves
  in-house, only `$lib/server/api.ts` changes; routes and URLs stay put.
- The `bff/` prefix marks proxy endpoints as frontend plumbing, distinct from a
  domain API the app would own.
- The client never holds the API token; the typeahead calls the guarded BFF, which
  attaches the token server-side.

## When to reconsider
If the app starts owning significant domain logic, or the round-trip through the
external API becomes the bottleneck, revisit whether some domain should move into
SvelteKit server routes (see the internal-API example).

## Score anchor
8 to 10 when secrets stay server-side and load is the single data seam. Dock
heavily if the token is exposed via `PUBLIC_` env or called from client code.

## References
- Server-only modules: https://svelte.dev/docs/kit/server-only-modules
- Load: https://svelte.dev/docs/kit/load
