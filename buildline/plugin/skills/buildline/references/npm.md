# npm: publish and promote

The registry plays the store: `ship` publishes under a prerelease dist-tag (default `beta`), and
`submit --confirm` moves the production dist-tag (default `latest`) to that exact version. It is a
tag move, never a re-publish.

## Prerequisites

- Node.js and a package manager (npm, pnpm, yarn, or bun; detected from the lockfile).
- `package.json` with `name` and `version`; the package identity comes from there.
- An npm **automation or granular token** with read-write access to the package or scope, stored
  as a secret reference (`keychain:buildline-npm-token`, `file:`, or `env:NPM_TOKEN`). BuildLine
  writes it only to a temporary `0600` `.npmrc` that it removes after the run.
- For a scoped package (`@you/lib`), a scope the user owns and `access: public`.

## Commands

| Command | What it does |
|---|---|
| `build` | install dependencies and run the `build` script (skip with `npm.build: false`). |
| `ship` | publish `<name>@<version>` under the `ship.tag`; refuses if that version is already published. **Ask before running**: a published version is public and can never be reused. |
| `submit` | preview moving `submit.tag` to the version; `--confirm` applies it. |

- Bump `version` in `package.json` before every `ship`.
- Provenance is added automatically when a CI OIDC issuer is present (GitHub Actions); locally it
  publishes without it and says so.
- `--cancel` is not supported. To undo a promotion: `npm dist-tag add <name>@<previous> latest`.
