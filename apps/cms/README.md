# cms

Headless, multi-tenant WordPress with GraphQL, for local development.

Multi-tenancy is **subdirectory Multisite**: one WordPress install and one
database, with an isolated site per tenant. Each tenant gets its own content
tables, so posts, media, menus and settings do not leak between them. WPGraphQL
is network-activated, so every tenant — including ones added later — exposes its
own `/graphql` endpoint automatically.

"Headless" here means nothing consumes it yet. `apps/web` is deliberately
untouched; this app just stands up the content API.

## Quick start

Requires Docker.

```sh
npm run up -w cms      # start (first run installs WordPress, ~1-2 min)
npm run setup -w cms   # follow the initializer log
```

Or from this directory: `npm run up && npm run setup`.

| What | Where |
| --- | --- |
| Admin (network) | http://localhost:8080/wp-admin — `admin` / `admin` |
| Primary tenant | http://localhost:8080/graphql |
| Tenant A | http://localhost:8080/tenant-a/graphql |
| Tenant B | http://localhost:8080/tenant-b/graphql |

Every tenant also serves REST at `<url>wp-json/wp/v2`.

```sh
curl -s -X POST http://localhost:8080/tenant-a/graphql \
  -H 'Content-Type: application/json' \
  -d '{"query":"{ generalSettings { title url } }"}'
```

Other scripts: `npm run down` (stop, keep data), `npm run logs`, and
`npm run clean` (stop and **delete both volumes** — full reset).

## Configuration

Copy `.env.example` to `.env`; Compose picks it up from this directory
automatically. Everything has a working default, so `.env` is optional.

The credentials here are throwaway local defaults. Don't reuse them anywhere
that matters, and note the stack binds to all interfaces, not just loopback.

## Tenants

Tenants come from `WORDPRESS_TENANTS`, a comma-separated list of `slug:Title`:

```sh
WORDPRESS_TENANTS=tenant-a:Tenant A,tenant-b:Tenant B
```

Add one by appending to the list and re-running `npm run up`. The initializer is
idempotent and matches existing sites **by path**, so re-runs skip sites that
already exist and titles you've since edited in the admin are preserved.

Removing a slug from the list does **not** delete the site — use
`wp site delete` in the admin or via wp-cli.

## Changing the port

Subdirectory Multisite stores each tenant's domain (including the port) in the
`wp_blogs` table at install time, so `WORDPRESS_PORT` is **not** safe to change
after the first `up`. The default is `8080`, chosen because port 80 is commonly
taken by other local stacks.

To change it, either reset with `npm run clean` and set both `WORDPRESS_PORT`
and `WORDPRESS_SITE_URL` before the next `up`, or search-replace the stored
domain:

```sh
docker compose run --rm wp-cli wp search-replace \
  'localhost:8080' 'localhost:9090' --network --skip-columns=guid
```

## GraphQL CORS

WPGraphQL's default is `Access-Control-Allow-Origin: *`, which allows any page
to read the endpoint from a visitor's browser. `mu-plugins/headless-cors.php`
replaces that with an allow-list from `CMS_GRAPHQL_ALLOWED_ORIGINS`
(default: `http://localhost:3000`).

This governs browser access only — CORS is enforced client-side, so server-side
callers are unaffected regardless. It is not an authentication boundary; all
content exposed over `/graphql` is public either way. Delete the file to return
to WPGraphQL's open default.

## Notes

- WordPress core is pinned to `7.1.3` in two places that must stay in step: the
  `wordpress` image tag and `WORDPRESS_CORE_VERSION`. The official image only
  seeds core into an *empty* volume, so the initializer runs `wp core update`
  explicitly when an existing install trails a bumped tag.
- `mu-plugins/` is bind-mounted, so edits apply immediately with no rebuild.
  Must-use plugins activate network-wide and can't be deactivated from the
  admin — delete the file instead.
- This app has no `build`, `dev` or `lint` script on purpose, so `turbo run`
  from the repo root never tries to start Docker as part of a build.
- The `.htaccess` for subdirectory Multisite is written by the initializer;
  wp-cli can't regenerate it, and tenant paths 404 without it.
