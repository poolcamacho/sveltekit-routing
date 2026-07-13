# Example: Localized Public Website

**Profile.** A public, indexable content site (marketing, docs, blog) served in
several languages. SEO and shareable per-language URLs matter.

**Routing approach.** Locale prefix in the URL via an optional `[[lang]]` segment,
so the default language serves at the root and other languages get an explicit
prefix. Content is prerendered where possible, with `hreflang` and canonical tags.
Locale negotiation alone would be wrong here because crawlers need distinct,
indexable URLs per language; see `references/architecture-decisions.md`.

## Route tree

```
src/routes/
├── [[lang=locale]]/                 # optional locale: '' (default) or 'fr','de'
│   ├── +layout.server.ts            # resolve + validate locale; load messages
│   ├── +layout.svelte               # sets <html lang>, renders hreflang links
│   ├── +page.svelte                 # /  and  /fr
│   ├── blog/
│   │   ├── +page.svelte             # /blog  and  /fr/blog
│   │   └── [slug]/+page.ts          # /blog/x  and  /fr/blog/x
│   └── docs/[...path]/+page.ts      # nested docs tree, catch-all scoped here
├── sitemap.xml/+server.ts           # enumerate canonical per-locale URLs
├── src/params/locale.ts             # matcher: only supported locales
└── src/params/slug.ts
```

## Key code

Validate the locale and expose it to the subtree:

```ts
// src/routes/[[lang=locale]]/+layout.server.ts
import { error } from '@sveltejs/kit';
const SUPPORTED = ['fr', 'de'];           // '' at root means default locale
export const load = async ({ params }) => {
  const lang = params.lang ?? 'en';
  if (params.lang && !SUPPORTED.includes(params.lang)) throw error(404);
  return { lang, messages: await loadMessages(lang) };
};
```

```ts
// src/params/locale.ts
export function match(param: string) {
  return ['fr', 'de'].includes(param); // '' (absent) handled by optionality
}
```

Prerender content and emit canonical + hreflang in the layout:

```ts
// src/routes/[[lang=locale]]/blog/[slug]/+page.ts
export const prerender = true; // static content, same for everyone
```

## Why this shape
- `[[lang=locale]]` gives `/blog/x` for the default language and `/fr/blog/x` for
  French, both indexable and shareable, without duplicating the tree per language.
- The matcher rejects unknown locales as 404s, so `/xx/blog` does not silently
  render.
- The docs `[...path]` catch-all is scoped under `docs/`, so it cannot swallow
  sibling top-level routes.
- Prerendering plus canonical and `hreflang` tags gives correct SEO across
  languages.

## When negotiation would be better
If this were an authenticated app with no SEO need, drop the prefix and resolve
locale from the session or `Accept-Language` in hooks, keeping URLs locale-free.

## Score anchor
8 to 10 when locales are validated, canonical/hreflang are emitted, and content
prerenders. Dock for unvalidated locale params or duplicate-content URLs.

## References
- Advanced routing (optional segments, matchers): https://svelte.dev/docs/kit/advanced-routing
- Page options (prerender): https://svelte.dev/docs/kit/page-options
