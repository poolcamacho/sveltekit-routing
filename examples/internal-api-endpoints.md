# Example: SvelteKit Application with Internal API Endpoints

**Profile.** SvelteKit owns the domain and the data. Most mutations are in-app
forms, but the app also exposes internal API endpoints for a mobile client, a
webhook receiver, and some client-driven interactions. One deployable, one auth
story, shared types.

**Routing approach.** Pages use form actions for progressively enhanced in-app
mutations. `+server.ts` endpoints exist only for machine clients and non-page
responses, and every endpoint calls the same `$lib/server/` services the form
actions do, so the domain logic has one home. See
`references/architecture-decisions.md` for pages-vs-endpoints.

## Route tree

```
src/routes/
├── (app)/
│   ├── +layout.server.ts             # session guard (shell-wide)
│   ├── todos/
│   │   ├── +page.svelte
│   │   └── +page.server.ts           # form actions: create/toggle/delete
│   └── settings/+page.server.ts
├── api/
│   ├── todos/
│   │   ├── +server.ts                # GET list, POST create (mobile client)
│   │   └── [id=uuid]/+server.ts      # GET, PATCH, DELETE one
│   └── webhooks/
│       └── github/+server.ts         # POST only; signature-verified
└── src/params/uuid.ts
```

## Key code

One service, two surfaces. Form action and endpoint both call it:

```ts
// src/lib/server/todos.ts   (the single implementation)
export async function createTodo(userId: string, input: unknown) {
  const data = TodoSchema.parse(input);          // validation lives with the domain
  return db.todo.create({ userId, ...data });
}
```

```ts
// src/routes/(app)/todos/+page.server.ts   (in-app form, progressive enhancement)
import { createTodo } from '$lib/server/todos';
export const actions = {
  create: async ({ request, locals }) => {
    const form = await request.formData();
    await createTodo(locals.user.id, Object.fromEntries(form));
    return { ok: true };
  }
};
```

```ts
// src/routes/api/todos/+server.ts   (machine client; same service)
import { createTodo } from '$lib/server/todos';
export const POST = async ({ request, locals }) => {
  if (!locals.user) throw error(401);
  return json(await createTodo(locals.user.id, await request.json()), { status: 201 });
};
export const GET = async ({ locals }) => {
  if (!locals.user) throw error(401);
  return json(await listTodos(locals.user.id));
};
```

Webhook verified before any work:

```ts
// src/routes/api/webhooks/github/+server.ts
export const POST = async ({ request }) => {
  const body = await request.text();
  if (!verifySignature(request.headers, body)) throw error(401);
  await handleGithubEvent(JSON.parse(body));
  return new Response(null, { status: 204 });
};
```

## Why this shape
- Domain logic and validation live in `$lib/server/todos.ts`; the form action and
  the endpoint are thin adapters, so there is no divergence between the web and
  mobile paths.
- Endpoints handle HTTP methods explicitly and guard on the server, matching the
  page guard, so `api/todos` is not a hole around the guarded page.
- The webhook verifies its signature before doing any work, since it is a public
  entry point.
- Ids in `api/todos/[id=uuid]` are matcher-validated; ownership is still checked in
  the handler.

## When to version
These endpoints are first-party and deployed with the app, so they stay
unversioned. If an external consumer you do not control appears, introduce a
version boundary and keep the shared service. See `references/url-design.md`.

## Score anchor
8 to 10 when endpoints and form actions share services and both guard on the
server. Dock for domain logic duplicated between the action and the endpoint, or an
unguarded endpoint beside a guarded page.

## References
- Form actions: https://svelte.dev/docs/kit/form-actions
- Routing (`+server`): https://svelte.dev/docs/kit/routing
