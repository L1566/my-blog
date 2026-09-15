## Development

When starting the dev server, use background mode:

```
astro dev --background
```

Manage the background server with `astro dev stop`, `astro dev status`, and `astro dev logs`.

## Deployment

The site is fully prerendered and deployed to Cloudflare Workers as static assets. `wrangler.jsonc` sets `assets.directory` to `./dist` and deliberately has no `main`, so only assets are uploaded.

- `npm run deploy` — `astro build && wrangler deploy`; `npm run preview:worker` — `astro build && wrangler dev`.
- Cloudflare Workers Builds (Git integration) uses build command `npx astro build` and deploy command `npx wrangler deploy`.
- The `@astrojs/cloudflare` adapter is intentionally absent. Add it with `npx astro add cloudflare` only when on-demand rendering or Cloudflare bindings are needed, together with the `main` entry point documented in Cloudflare's Astro guide.
- `not_found_handling: '404-page'` serves `dist/404.html`, generated from `src/pages/404.astro`; keep that page when the 404 routing behavior matters.
- Cloudflare Workers Builds installs with the build image's npm 10.9.2 (`npm ci`). Local npm 11 drops the wasm32 optional chain (`@emnapi/core`, `@emnapi/runtime`) required by `sharp` and `@astrojs/*` from the lockfile, and `npm ci` then fails with `Missing: @emnapi/runtime@1.11.3 from lock file`. Regenerate lockfile changes with `npx npm@10.9.2 install --package-lock-only` and confirm with `npx npm@10.9.2 ci --dry-run` before committing.

## Documentation

Full documentation: https://docs.astro.build

Consult these guides before working on related tasks:

- [Adding pages, dynamic routes, or middleware](https://docs.astro.build/en/guides/routing/)
- [Working with Astro components](https://docs.astro.build/en/basics/astro-components/)
- [Using React, Vue, Svelte, or other framework components](https://docs.astro.build/en/guides/framework-components/)
- [Adding or managing content](https://docs.astro.build/en/guides/content-collections/)
- [Adding styles or using Tailwind](https://docs.astro.build/en/guides/styling/)
- [Supporting multiple languages](https://docs.astro.build/en/guides/internationalization/)
