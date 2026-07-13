# Version Verification

## Verification date and method

- Verification date: 2026-07-13 (current calendar year 2026).
- Method: official documentation and repositories checked via the web (see
  `references/official-sources.md`).

If documentation access is unavailable at run time, the agent must re-verify and
must not claim current-year verification. Report the version actually observed in
the project's `package.json` and note that live docs could not be confirmed.

## Verified stable versions

- Svelte: 5.56.4
- SvelteKit (`@sveltejs/kit`): 2.69.2
- Build tooling: Vite 8 (SvelteKit requires `vite ^8.0.12`).

Do not hardcode these numbers inside prose recommendations. Prefer phrasing like
"in the current stable SvelteKit". Use the concrete numbers only here and, if
useful, in README badges. Always confirm the version the project actually uses in
its `package.json` before giving version-sensitive routing advice.

## Classification relevant to routing

### Stable (recommend freely when they fit)

- File-based routing: `+page.svelte`, `+page.ts`/`.js`, `+page.server.ts`/`.js`,
  `+layout.svelte`, `+layout.ts`, `+layout.server.ts`, `+server.ts`,
  `+error.svelte`.
- Dynamic `[param]`, optional `[[param]]`, and rest `[...param]` segments.
- Parameter matchers in `src/params/`.
- Route groups `(name)/` and layout resets `+layout@` / `+page@`.
- Nested layouts and layout load composition.
- Load functions (universal and server), form actions, and hooks
  (`hooks.server.ts`, `hooks.client.ts`, `hooks.ts`).
- Page options: `prerender`, `ssr`, `csr`, `trailingSlash`.
- `$env` modules (with the `PUBLIC_` prefix rule) and `$app/state`.
- Deployment adapters.
- Reactivity via runes (`$state`, `$derived`, `$effect`, `$props`, `$bindable`),
  usable in `.svelte` and `.svelte.ts` files.

### Experimental (opt-in; never recommend as a production default)

- Remote functions (`query`, `form`, `command`, `prerender` in `.remote.ts`),
  available behind a config flag since SvelteKit 2.27. They can call server logic
  without a `+server.ts` endpoint, which touches routing architecture, but they
  are experimental. Present them only as a labeled experimental alternative, and
  keep endpoint/form-action guidance as the default.

### Deprecated (flag and migrate away)

- `$app/stores` (including `page` and `navigating` from it) is superseded by
  `$app/state`, which requires runes, and is subject to removal in a future major.
  In routing code that reads the current page or navigation state, prefer
  `$app/state`.

## Testing tools (for verifying routing behavior)

- Vitest for unit and component tests, with Vitest browser mode via
  `vitest-browser-svelte` for component tests.
- Playwright for end-to-end tests, which is the reliable way to verify precedence,
  guards, redirects, and trailing-slash behavior against real navigation.
