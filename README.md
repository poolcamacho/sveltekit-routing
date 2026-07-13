# SvelteKit Routing Skill

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Install with skills](https://img.shields.io/badge/install-npx%20skills%20add-black)](https://skills.sh)
[![Built for Claude Code](https://img.shields.io/badge/built%20for-Claude%20Code-6f42c1)](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview)
[![GitHub stars](https://img.shields.io/github/stars/poolcamacho/sveltekit-routing?style=social)](https://github.com/poolcamacho/sveltekit-routing/stargazers)

A reusable [Agent Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview)
that turns Claude into a senior SvelteKit architect for routing. It helps design a
new route tree, implement routes correctly, review an existing routing
architecture, detect routing anti-patterns, and produce a concrete, tradeoff-aware
improvement and migration plan.

It is deliberately not a beginner tutorial and not a restatement of the docs. It
is opinionated where SvelteKit and security demand it (server-enforced auth, the
URL as a public contract, precedence determinism) and non-dogmatic everywhere
else. Every recommendation comes with why it exists, when to apply it, when not
to, and the alternatives.

## What it does

- Design from scratch. Recommends the simplest routing architecture that fits the
  app's URL semantics, security surface, rendering needs, and team, and explains
  when to add a locale or version axis and when not to.
- Review an existing route tree. Runs a fourteen-step audit and emits a fifteen-
  section report: executive summary, a routing score (1 to 10), current route tree
  analysis, strengths, problems and risks, security concerns, URL design concerns,
  immediate and medium-term improvements, a suggested route tree, a migration plan,
  a verification checklist, the official references consulted, the documentation
  verification date, and the verified stable SvelteKit version.
- Detect problems. Deep nesting without a product reason, fat layouts, client-only
  authorization, business logic in route files, duplicate or missing guards,
  cosmetic route groups, unstable URLs, leaked internal ids, catch-all overreach,
  endpoint/service mixing, unversioned APIs, redirect loops, inconsistent trailing
  slash, precedence mistakes, and more, each with a detection heuristic and a fix.
- Explain tradeoffs. Flat vs nested, URL vs code hierarchy, groups vs nesting,
  shared vs isolated layouts, pages vs endpoints, internal vs external backend,
  central vs per-route authorization, tenant in host vs path vs session, locale
  prefixes vs negotiation, and route logic vs services.

## When it triggers

Laying out routes for a new SvelteKit app, asking where an endpoint or page should
live, asking about `+page`/`+layout`/`+server` files, dynamic, optional, or rest
parameters, matchers, route groups, nested layouts or layout resets, public vs
protected routes, authentication-aware routing, authorization boundaries, multi-
tenant routing, locale-aware routes, versioned APIs, SEO-friendly or canonical
URLs, route precedence and conflicts, redirects, trailing slash, or
prerender/SSR/CSR implications. It triggers even when the word "routing" is not
used, for questions like "why is this URL resolving to the wrong page?" or "should
this be a route group or a folder?".

## Contents

```
sveltekit-routing/
├── SKILL.md                          # entry point: role, workflow, output format
├── README.md                         # this file
├── references/
│   ├── routing-mechanics.md          # special files, params, matchers, groups, resets, precedence
│   ├── architecture-decisions.md     # the routing forks, with tradeoffs
│   ├── anti-patterns.md              # smells, detection, remediation
│   ├── auth-and-boundaries.md        # public/protected, authz, multi-tenant
│   ├── url-design.md                 # SEO, canonical, naming, locales, versioning
│   ├── review-checklist.md           # the audit checklist with heuristics and fixes
│   ├── official-sources.md           # docs and repos consulted
│   └── version-verification.md       # verification date, versions, classification
└── examples/
│   ├── small-saas.md                 # annotated route trees, one per archetype
│   ├── dashboard.md
│   ├── multi-tenant-saas.md
│   ├── localized-public-website.md
│   ├── enterprise.md
│   ├── frontend-separate-backend.md
│   └── internal-api-endpoints.md
```

Claude loads `SKILL.md` when the skill triggers, then pulls in the specific
reference or example it needs, so the deep material stays out of context until it
is relevant.

## Design principles

1. The URL is a product contract. URLs are public, linked, and indexed; they
   should name durable product concepts, not code or UI structure. A URL change is
   a migration.
2. The boundary is on the server. Authentication, authorization, and tenancy are
   enforced in server code, never by hiding a link or a client check.
3. Fit over dogma. No single route tree is universally correct. Recommendations
   are calibrated to size, team, and trajectory.
4. Evidence-based reviews. Findings cite real paths and are ordered by impact, and
   the score is anchored to a rubric.
5. Current and verifiable. The skill instructs Claude to confirm routing behavior
   against the official docs and labels features as stable, experimental (for
   example remote functions), or deprecated (for example `$app/stores`).

## Official documentation

- SvelteKit routing: https://svelte.dev/docs/kit/routing
- Advanced routing: https://svelte.dev/docs/kit/advanced-routing
- Server-only modules: https://svelte.dev/docs/kit/server-only-modules

## Installation

Install with the [skills](https://skills.sh) CLI, which works across Claude Code
and other agents:

```bash
npx skills add poolcamacho/sveltekit-routing
```

The command clones this repo, detects the skill from `SKILL.md`, and lets you
choose which agents to install it into (Claude Code and the universal target are
selected by default).

You can also install it manually by copying the `sveltekit-routing` folder into
your agent's skills directory (for Claude Code, `.claude/skills/`).

## Usage

Once installed, ask naturally, for example "review the routing of my SvelteKit
app", "should tenant id go in the path or the session?", or "design the route tree
for a localized marketing site". The agent will consult this skill and respond as
an architect, using the report format for reviews.

## License

Released under the [MIT License](LICENSE). Free to use, adapt, and redistribute
with attribution.
