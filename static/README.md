# Static Pages

This folder contains starter static pages referenced by `custom-values.yaml`:

- `pages/home.html`
- `pages/welcome.html`
- `pages/rules.html`
- `assets/logo.svg`
- `assets/welcome-bg.svg`

Expected public URLs:

- `https://element.danishkodemonkey.net/pages/home.html`
- `https://element.danishkodemonkey.net/pages/welcome.html`
- `https://element.danishkodemonkey.net/pages/rules.html`
- `https://element.danishkodemonkey.net/static-assets/logo.svg`
- `https://element.danishkodemonkey.net/static-assets/welcome-bg.svg`

Notes:

- The chart can now serve these pages directly with in-cluster Nginx:
  - set `element.staticPages.enabled: true`
  - route path prefix with `element.staticPages.pathPrefix` (default `/pages`)
- You can still host these files externally (CDN/object storage) if preferred.
- Keep URLs in `custom-values.yaml` aligned with where you publish/serve them.
