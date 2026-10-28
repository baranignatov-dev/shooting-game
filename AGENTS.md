# Setup notes (Base44 sandbox)

This repo is a single-file static browser game: `index.html` (HTML + CSS + inline JS canvas game).
There is no build step, no package manager, no backend, and no external services or credentials.

## Running it

```bash
docker compose -f docker-compose.base44.yml up -d
```

The `web` service serves the repo on host port 3000:
`python3 -m http.server 80 --directory /site`, with the checkout bind-mounted read-only at `/site`.
Files are read per request, so edits appear on the next reload (no rebuild, no watcher needed).

## Non-obvious findings

- **Do not serve this with the stock `nginx:alpine` image.** The sandbox checkout is root-owned with
  mode `0700`, and nginx spawns workers as the unprivileged `nginx` user, which cannot traverse the
  bind-mounted directory — every request returns `403 Forbidden`. Any server used here must run as
  root (as `python:3.12-slim` does by default), or the checkout permissions must change.
- Because there is no live-reload dev server, call `reload_preview` after editing `index.html`.
- The healthcheck uses Python's `urllib` (the image has no curl/wget) and probes `/index.html`.

## Verifying it works

- `curl -s -o /dev/null -w '%{http_code}' http://localhost:3000/` must return `200`.
- In the browser: HUD reads `Score: 0` / `Health: 100`, Spider-Man is drawn at center, orange enemy
  squares drift toward him. Clicking anywhere fires a web shot; hitting an enemy adds 10 to the score.
- Note on gameplay: each colliding enemy removes 1 HP **every frame**, so an idle player reaches
  `GAME OVER!` in roughly 3-5 seconds. That is the game's own balance, not a deployment problem.
