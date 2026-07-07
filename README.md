# The Mystical Wild

Source for [krishnaswild.com](https://krishnaswild.com) — Krishna's wildlife photography
portfolio. Single-page site: `index.html` (inline CSS/JS), deployed on GitHub Pages via
`CNAME`.

## Structure

- `index.html` — the entire site. Content lives in JS config objects near the bottom
  of the file:
  - `CHAPTERS` — the three story chapters and their scenes (image, caption, copy).
  - `PRINTS` — the prints-wall items and their detail-page sizes/materials/pricing.
  - `PRICING` — shared pricing constants (edition strategy, FX rates) referenced by `PRINTS`.
  Editing these objects is enough to retitle, reorder, or reprice content — no markup changes
  needed.
- `images/` — all content photos, WebP only (each `CHAPTERS`/`PRINTS` entry references a
  `.webp` file there). The exception is `images/print-the-promise.jpg`, kept for the
  `og:image` meta tag since some social-media crawlers don't render WebP previews.
  Note: a `PRINTS` entry's `img` filename (minus extension) doubles as its URL slug —
  `images/print-the-promise.webp` routes to `#print-the-promise` — so the slug is derived
  from the basename only, decoupled from the `images/` path.
- `favicon.svg` / `favicon.png` / `apple-touch-icon.png` — site icons, referenced via
  `<link>` tags in `<head>`.

## Local development

No build step. Serve the directory with any static file server, e.g.:

```
python3 -m http.server 8000
```

then open `http://localhost:8000`.

## Deployment

Push to the branch GitHub Pages is configured to serve (see repo Settings → Pages). The
`CNAME` file points the custom domain at Pages.
