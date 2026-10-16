# SOYO — working notes

Next.js 14 app-router app (`app/`) that lists video files from one directory and
streams them. No database, no separate backend — the API is Next route handlers
under `app/api/`.

## Run it here

```bash
docker compose -f docker-compose.base44.yml up -d --build   # http://localhost:3000
docker compose -f docker-compose.base44.yml logs -f web      # dev-server output
```

One `web` service: `node:20-bookworm-slim`, source bind-mounted at `/app`, runs
`npm ci` (only when `node_modules` is missing or stale) and then
`next dev -H 0.0.0.0 -p 3000`. Edits hot-reload; a dependency change needs
`up -d` again so `npm ci` runs.

## Video library

`config.json` is tracked and holds the author's local path (`F:/`), so it is
**not** rewritten for the sandbox. Instead `readConfig()` in
`app/lib/serverUtils.ts` falls back to `MOVIE_DIR` when the configured
directory does not exist on this machine — the compose file sets
`MOVIE_DIR=/media`, bind-mounting `./.base44/media` (git-ignored) there.

- Sample clips + extracted poster frames live in `./.base44/media/` and
  `./.base44/media/thumbnails/`; the directory is created by Docker if missing,
  and an empty library simply renders an empty list.
- `readConfig()` precedence: an existing `config.json` directory wins (a local
  `F:/` run is unaffected), then `MOVIE_DIR`, then the built-in defaults.
- The in-app folder button calls `POST /api/movies`, which **writes
  `config.json`** — it modifies the repo file, so revert it if you only meant
  to try a path out.
- AniList thumbnails are fetched with no key; cached posters are written into
  `<MOVIE_DIR>/thumbnails/<name-without-extension>.jpg`.
  `IMDB_API_KEY` (OMDb, optional) only adds IMDb posters — the app runs without
  it.

## Verifying it works

- `GET /api/movies` → JSON array with one entry per clip (500 if the library
  directory is unreadable).
- `GET /api/thumbnail/<file>.mp4` → `image/jpeg` (local poster, remote poster,
  else `public/default.jpg`).
- `GET /api/stream/<file>.mp4` → `206` with `Content-Range` when a `Range`
  header is sent.
- Home page `/` renders the poster rows; `/stream/<file>.mp4` plays the clip.
- Compose healthcheck: `GET /` on port 3000 must return 200 (dev server has to
  compile the first request, hence the 120s start period).
