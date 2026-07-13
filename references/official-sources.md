# Official Sources Consulted

The routing guidance in this skill is grounded in the official SvelteKit and
Svelte documentation and repositories. Consult these directly when giving
version-sensitive advice, and record which pages you used in section 13 of a
review report. If documentation access is unavailable at run time, re-verify
before claiming current-year accuracy.

## Documentation

- https://svelte.dev/docs/kit
  SvelteKit documentation home. Entry point for routing, load, hooks, adapters.
- https://svelte.dev/docs/kit/routing
  Core file-based routing: `+page`, `+layout`, `+server`, `+error`, load, actions.
- https://svelte.dev/docs/kit/advanced-routing
  Rest and optional params, parameter matchers, route groups, layout resets
  (`@`), and route sorting/precedence rules.
- https://svelte.dev/docs/kit/load
  Universal vs server load, layout load composition, invalidation, dependencies.
- https://svelte.dev/docs/kit/form-actions
  Form actions on `+page.server.ts`, the page-vs-endpoint decision for mutations.
- https://svelte.dev/docs/kit/page-options
  `prerender`, `ssr`, `csr`, `trailingSlash`, and how they cascade to children.
- https://svelte.dev/docs/kit/hooks
  `handle`, `handleError`, `handleFetch`; the place for central auth and tenancy.
- https://svelte.dev/docs/kit/server-only-modules
  The build-enforced server/client boundary that makes auth decisions safe.
- https://svelte.dev/docs/kit/adapters
  Deployment adapters and their effect on prerendering, edge/serverless routing,
  and host-based (multi-tenant, per-locale-domain) routing.
- https://svelte.dev/docs/kit/state-management
  Per-request state scoping; why per-user state must not live in module scope.
- https://svelte.dev/docs/svelte
  Svelte language docs, including runes, the current reactivity model.

## Repositories and release notes

- https://github.com/sveltejs/kit
  SvelteKit source, issues, and the `@sveltejs/kit` changelog for routing changes.
- https://github.com/sveltejs/svelte
  Svelte source and changelog.
- https://github.com/sveltejs/kit/releases
  Release notes; check for routing, precedence, and page-option changes per major.
- https://svelte.dev/docs/kit/migrating
  Migration guidance between SvelteKit majors, including routing-related changes.
