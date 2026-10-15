# super-rotary-phone

A [Turborepo](https://turbo.build) monorepo using npm workspaces.

## Layout

| Path | What it is |
| --- | --- |
| [`apps/web`](apps/web) | Next.js 16 frontend — TypeScript, ESLint, App Router |
| [`apps/cms`](apps/cms) | Headless multi-tenant WordPress + GraphQL, via Docker — see its [README](apps/cms/README.md) |
| [`index.js`](index.js) | Schedule feed fetcher (see [below](#schedule-fetcher)) |

Workspaces are `apps/*` and `packages/*`. There is no `packages/` yet.

## Requirements

- Node 22+ — `index.js` relies on the built-in `fetch`
- npm 10+ (the repo is npm-based; there's no pnpm or yarn lockfile)
- Docker, for `apps/cms` only

## Getting started

```sh
npm install        # installs every workspace
npm run dev        # turbo: starts apps/web on :3000
npm run build      # turbo: builds every workspace that has a build task
npm run lint       # turbo: lints every workspace that has a lint task
```

`apps/cms` is excluded from `dev`, `build` and `lint` on purpose — it's a Docker
stack, not something that should start as a side effect of a build. Drive it
directly instead:

```sh
npm run up -w cms      # start WordPress (first run takes a minute or two)
npm run setup -w cms   # follow the initializer log
npm run down -w cms    # stop, keeping data
npm run clean -w cms   # stop and delete the volumes (full reset)
```

Once up, the network admin is at http://localhost:8080/wp-admin (`admin` /
`admin`) and each tenant serves its own GraphQL endpoint. Port, tenants and
credentials are all configurable — see [`apps/cms/README.md`](apps/cms/README.md).

## Schedule fetcher

`index.js` fetches the TRUE CRIME EPG feed (Pawa.tv `channel_24`) and prints it
to stdout as JSON — roughly a thousand events, so redirect it somewhere:

```sh
npm start > schedule.json
```

Two quirks of that feed are handled in the script: it's served with a UTF-8 BOM
that the XML parser rejects as stray content, and the `programmeImage`,
`seriesImage` and `episodeImage` tags each appear twice per event, as
`type="original"` and `type="cropped"` (preserved as `@type`).

The fetcher still lives at the repo root rather than in a workspace, which is
why the root `package.json` carries both the `fast-xml-parser` dependency and a
`start` script. Moving it under `packages/` is a known tidy-up.

## Caveats

- **`apps/cms` port is 8080, not 80.** Subdirectory Multisite records each
  tenant's domain *and port* in the database at install time, so it can't be
  changed afterwards without a search-replace. Details in the CMS README.
- **`npm audit` reports high-severity findings** from `braces`, reaching in via
  `eslint-config-next`. It's lint-time only and never enters a production
  bundle; `npm audit fix --force` would downgrade `eslint-config-next` several
  major versions, which is the worse trade. It ships this way from
  `create-next-app`, so it's upstream.
