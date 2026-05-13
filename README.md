# Run a Call API docs

Public documentation for the Run a Call External REST API. Hosted at
**[docs.runacall.com](https://docs.runacall.com)** via GitHub Pages.

## What's here

- `openapi.json` — OpenAPI 3.1 spec, regenerated from the main repo's
  Zod schemas via `zod-to-openapi`.
- `postman-collection.json` — Postman v2.1 collection generated from
  the same spec via `openapi-to-postmanv2`. Linked from the top-right
  of `index.html` as a one-click download.
- `index.html` — Redoc CDN bundle. Renders `openapi.json` into a
  three-panel API reference + the webhooks section.
- `CNAME` — GitHub Pages custom domain config (`docs.runacall.com`).

## Updating the spec

This repo holds a static snapshot of the API surface. The source of
truth lives in the main app repo (`product-hvac`). To sync both
artifacts (OpenAPI + Postman):

```bash
# in product-hvac:
npm run gen:spec    ../runacall-api-docs/openapi.json
npm run gen:postman ../runacall-api-docs/postman-collection.json

# back here:
git add openapi.json postman-collection.json
git commit -m "sync spec from product-hvac@<short-sha>"
git push
```

GitHub Pages auto-deploys on push to `main`.

## No CI for now

We do manual sync on each release. When release cadence picks up, we
can add a GitHub Action on the main repo that pushes the spec here
automatically.

## Stack

- **Renderer:** Redoc 2.x via CDN (`cdn.redocly.com`) — supports
  OpenAPI 3.1 + the top-level `webhooks:` block.
- **Hosting:** GitHub Pages from `main` branch root.
- **DNS:** `docs.runacall.com` CNAME → `<user>.github.io` (see DNS
  provider settings).

## License

The spec describes a proprietary API surface — see the main app's
license. The site infrastructure (index.html, README) is MIT.
