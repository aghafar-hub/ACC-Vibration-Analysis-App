# Development, build & deployment

## Local development

```bash
npm install
npm run dev
```

Opens a dev server (default `http://localhost:5173`). The app ships with a
working default webhook URL (the production endpoint the original bundle
had hardcoded — see "Environment / secrets" below), so it'll try to sync
against real data immediately. To point at a different sheet, go to
**Settings → Configuration** (pass key `17593` — see
[API_CONTRACT.md](./API_CONTRACT.md#known-gaps) for why that's not real
security) and change the Webhook URL — it's stored in your browser's
`localStorage` only, never in the repo or a `.env` file.

## Quality checks

```bash
npm run lint          # ESLint (flat config, eslint.config.js)
npm run format        # Prettier — auto-fix formatting
npm run format:check  # Prettier — check only, no changes (used in CI)
npm run build          # Production build via Vite
```

All four run in CI (`.github/workflows/ci.yml`) on every push and pull
request against `main`.

## Production build

```bash
npm run build
```

Outputs a static site to `dist/` — an `index.html` and a hashed JS bundle
in `dist/assets/`. This is a fully static site; it can be hosted anywhere
that serves static files (GitHub Pages, Netlify, S3 + CloudFront, an
internal web server, etc.). There is no server-side code to deploy — all
data access happens client-side against the Apps Script webhook.

### Base path

`vite.config.js` sets:

```js
base: "/acc-vibration-analysis-app/";
```

This matches being served from a subpath like
`https://<user>.github.io/acc-vibration-analysis-app/`. If you deploy to a
custom domain or a host's root path instead, change this to `base: "/"`
before building — otherwise asset URLs will 404.

### Deploying to GitHub Pages

`.github/workflows/deploy.yml` builds and publishes `dist/` via GitHub's
official Pages actions on every push to `main`. To enable it:

1. In the repo's **Settings → Pages**, set the source to **GitHub
   Actions**.
2. Push to `main` — the workflow builds and deploys automatically.
3. Confirm `vite.config.js`'s `base` matches the resulting URL path.

## Environment / secrets

There are none. The webhook URL is not a secret in the traditional sense —
Apps Script Web Apps deployed "Execute as: Me / Who has access: Anyone" are
meant to be reachable by anyone with the link, and the original app shipped
its production URL hardcoded directly in the client bundle (visible to
anyone who opened devtools). This rebuild keeps that same default
(`DEFAULT_WEBHOOK_URL` in `config.js`) rather than removing it, since
that's a straight reconstruction of the original's actual behavior, not
something this rebuild introduced. Nothing in this repo needs a `.env`
file, and none should be added unless a genuine secret is introduced later.
