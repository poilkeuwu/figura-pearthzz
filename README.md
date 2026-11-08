# figura.pearthzz — Figura backend on Cloudflare

Custom backend for the [Figura](https://github.com/FiguraMC/Figura) Minecraft mod with a **10 MB** avatar limit
(official: 1 MB). Cloudflare Worker + Durable Object (SQLite) + R2.

| Piece | Role |
|---|---|
| Worker (`src/index.ts`) | routing, public avatar download from R2, `/api/assets` proxy to the official backend |
| Durable Object `Hub` (`src/hub.ts`) | Mojang auth, tokens, avatar metadata, equip, WebSocket pings/events |
| R2 `figura-pearthzz-avatars` | avatar files |

## Deploy

```bash
npm install
npx wrangler login
npx wrangler r2 bucket create figura-pearthzz-avatars
npx wrangler deploy
```

Figura always connects with **HTTPS on a real hostname**, so attach a domain you own
(uncomment `routes` in `wrangler.toml`, e.g. `figura.<your-domain>`), or use the
`figura-pearthzz.<account>.workers.dev` URL printed by `deploy`.

## Use in game

Figura → Settings → **Dev** → *Server IP* → enter the hostname only
(e.g. `figura.example.com`, no `https://`). Everyone who should see your avatar must set the same value.
Then upload as usual.

## Config (`wrangler.toml` `[vars]`)

- `MAX_AVATAR_SIZE` — bytes, default `10485760` (10 MB). The client reads this from `/api/limits`.
- `MAX_AVATARS` — avatars per player (3 → at most 30 MB/player in R2).
- `ALLOWED_UUIDS` — comma-separated whitelist (dashed UUIDs). **Set this if the backend is private**;
  empty means any Minecraft account with a valid session can use it.
- `PING_SIZE`, `PING_RATE`, `RATE_UPLOAD`, `RATE_DOWNLOAD`, `TOKEN_TTL_HOURS`, `MOTD`.

## Test locally

```bash
echo 'DEV_MODE=1' > .dev.vars   # skips Mojang check — local only, never deploy this
npx wrangler dev
node scripts/smoke.mjs          # 17 checks: auth, 10MB limit, equip, download, WS ping/event
```

## Notes / limits

- Avatar downloads are **public** (no token) — avatars are public in Figura anyway, and it keeps the Durable Object off the hot path.
- Everything stateful goes through one Durable Object: fine for friends/small community, not for thousands of concurrent players.
- No per-user daily upload quota yet; combine with `ALLOWED_UUIDS` if you don't want strangers filling R2.
- Players on this backend only see avatars of others on the same backend.
- Built from the Figura client source (`backend2/*`); protocol may change in future Figura versions.

