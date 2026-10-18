# keys-api

A small API and dashboard for managing credential pools. Bots get the least-recently-used working key,
then report when a provider rejects it. Built with Bun, Hono, SQLite, and Next.js.

Providers are configured in the database, so adding one does not require a code change.

## Quick start

### API

```sh
bun install
cp .env.example .env   # set the required values; see Environment below
bun run dev             # http://localhost:8787
```

Or run the API with Docker Compose:

```sh
cp .env.example .env    # fill in the required values first
docker compose up -d
```

### Dashboard

In another terminal:

```sh
cd dashboard
bun install
cp .env.example .env.local   # NEXT_PUBLIC_API_URL=http://localhost:8787
bun run dev
```

The dashboard runs on `http://localhost:3000`. Its origin must be listed in the API's
`NEXT_PUBLIC_ORIGIN` setting.

## Environment

| Variable              | Required | Purpose                                                                                                    |
| --------------------- | -------: | ---------------------------------------------------------------------------------------------------------- |
| `SESSION_SECRET`      |      Yes | Secret used to sign admin sessions. Use a long random value.                                               |
| `ADMIN_PASSWORD_HASH` |      Yes | Admin password hash in `scrypt:<saltHex>:<keyHex>` format. Generate it with the command in `.env.example`. |
| `NEXT_PUBLIC_ORIGIN`  |      Yes | Allowed dashboard origins, comma-separated, with no trailing slash.                                        |
| `PORT`                |       No | API port; defaults to `8787`.                                                                              |
| `DB_PATH`             |       No | SQLite file path; defaults to `data/keys.db`.                                                              |

Keep `.env` private. The dashboard's `NEXT_PUBLIC_API_URL` is a separate setting: it is the API's
base URL, with no trailing slash.

## Use the API

Create a bot token in the dashboard and set `$TOKEN` to it. Use the returned key `id` as `$KEY_ID`:

```sh
curl -H "Authorization: Bearer $TOKEN" http://localhost:8787/api/youtube
```

If a provider rejects a key, flag it so it is removed from rotation:

```sh
curl -X PATCH \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"status":"flagged","reason":"403 account banned"}' \
  "http://localhost:8787/api/youtube/$KEY_ID"
```

A `503` response with `no_working_keys` means the pool is empty. `refresh_failed` means the provider
could not renew an existing key. `GET /api/health` is public and can be used for health checks.

### Admin endpoints

The dashboard logs in via `POST /api/admin/login`; the API returns a secure session cookie. Admin routes:

| Endpoint                                    | Purpose                                  |
| ------------------------------------------- | ---------------------------------------- |
| `GET, POST /api/admin/providers`            | List or add providers                    |
| `PATCH, DELETE /api/admin/providers/:slug`  | Update or delete a provider and its keys |
| `GET, POST /api/admin/providers/:slug/keys` | List or add keys                         |
| `PATCH, DELETE /api/admin/keys/:id`         | Update or delete a key                   |
| `GET, POST /api/admin/tokens`               | List or create bot tokens                |
| `DELETE /api/admin/tokens/:id`              | Revoke a token                           |

`deezer` and `youtube` are included by default. Add providers in the dashboard by defining their fields;
those fields become the form used to add keys.

## Deploy: Docker API + Vercel dashboard

The API needs a persistent server and database volume. The dashboard can run as a separate Vercel
project.

### 1. Host the API

On a VPS, point `api.example.com` to the server and install Docker, Docker Compose, Nginx, and Certbot.
The Compose port is bound to localhost (`127.0.0.1:8787`), so Nginx can proxy requests without exposing
plain HTTP publicly.

Create an Nginx site at `/etc/nginx/sites-available/keys-api`:

```nginx
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:8787;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable the site and issue a TLS certificate (Debian/Ubuntu):

```sh
sudo ln -s /etc/nginx/sites-available/keys-api /etc/nginx/sites-enabled/keys-api
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d api.example.com
```

Allow inbound ports 80 and 443. Set the API's `.env` values for production:

- `SESSION_SECRET`: generate one with `openssl rand -hex 32`.
- `ADMIN_PASSWORD_HASH`: use the scrypt command in `.env.example`.
- `NEXT_PUBLIC_ORIGIN`: the exact dashboard origin, e.g. `https://dashboard.example.com`.

The Compose file uses the GHCR image by default. Publish it with the manual **docker** workflow in
GitHub Actions, then deploy:

```sh
docker compose pull
docker compose up -d
```

To build from source instead, change `image:` to `build: .` in `compose.yml` and run
`docker compose up -d --build`. Keep the `keys-data` volume persistent; it contains the database and
all managed credentials. Confirm `https://api.example.com/api/health` works over HTTPS.

### 2. Deploy the dashboard to Vercel

Create a Vercel project for this repository with **Root Directory** set to `dashboard`. Use the Next.js
preset and add this environment variable:

| Variable              | Value                                         |
| --------------------- | --------------------------------------------- |
| `NEXT_PUBLIC_API_URL` | `https://api.example.com` (no trailing slash) |

Deploy or redeploy after setting it; Next.js embeds this public value at build time. Add a stable domain
such as `dashboard.example.com`, then ensure the API's `NEXT_PUBLIC_ORIGIN` exactly matches it, including
the scheme and with no trailing slash.

Using dashboard and API subdomains of the same domain is recommended. Browsers may block the admin
session cookie when the dashboard and API are on unrelated sites. Vercel preview deployments need their
exact origins added to the API's comma-separated `NEXT_PUBLIC_ORIGIN`; use the stable domain for normal
admin access.

## Development checks

```sh
bun test
bun run check-types
```

Tests use an in-memory database and do not need network access or environment variables.
