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

## Fixes applied to the game

The 654b90b commit shipped only `index.html`, so there was nothing to serve and the app never started.
Two gameplay bugs from that commit also made the preview unplayable and were fixed in place:

- **Aim followed stale touch coordinates.** `shootWeb()` only read `touchX`/`touchY`, which are set by
  touch events, so a mouse click fired every web shot at the top-left corner (0,0). It now takes the
  pointer coordinates from the event (`clientX`/`clientY`).
- **Damage was applied every animation frame.** Standing in an enemy drained 1 HP per frame (~60 HP/s),
  so `GAME OVER!` arrived 3-5 seconds after load. Each enemy now damages at most once per 700 ms.

## Verifying it works

- `curl -s -o /dev/null -w '%{http_code}' http://localhost:3000/` must return `200`.
- In the browser: HUD reads `Score: 0` / `Health: 100`, Spider-Man is drawn at center, orange enemy
  squares drift toward him. Clicking a point fires a web shot toward that point; hitting an enemy adds
  10 to the score. Health drains a few HP per second instead of draining instantly.
- Enemies are drawn on top of the player, so an enemy sitting on Spider-Man hides the sprite — visual
  only, not a rendering bug.
