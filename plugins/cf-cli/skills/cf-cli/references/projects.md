# Projects, deploy, CI

Source: https://developers.cloudflare.com/cf/projects/index.md, /cf/get-started/index.md, /cf/ci/index.md, /cf/agents/index.md

## Setup

- `npm i -g cf` (yarn/pnpm/bun too). Installs `cf` and `cloudflare` (same CLI). Update: `npm i -g cf@latest`. Global `cf` runs the project's installed copy; `npx cf@<VERSION>` does not hand off.
- `cf auth login` (`--no-browser` on SSH, `--force` re-login), `cf auth whoami`. Separate from Wrangler login.
- Profiles: `cf auth create <name>`, `cf auth activate <name> [dir]` (binds dir tree), `cf auth deactivate [dir]`, `cf auth list`, `cf auth delete <name>`, `--profile <name>`. All refuse while `CLOUDFLARE_API_TOKEN` is set.
- Completion: `cf complete zsh >> ~/.zshrc` (bash, fish, powershell).
- Account cache: `cloudflare-account[-<PROFILE>].json` in `.cache/cloudflare/` under nearest `node_modules`, else `.cloudflare/cache/`. Delete to re-choose; `cf auth login --force`/`logout` also clear it unless `CLOUDFLARE_API_TOKEN` set.
- `.env` in cwd: only `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_ZONE_ID`, `CLOUDFLARE_COMPLIANCE_REGION`, `CLOUDFLARE_ACCESS_CLIENT_ID`, `CLOUDFLARE_ACCESS_CLIENT_SECRET`. Not `.env.local`/`.env.<mode>`. Real env wins. `--local` skips it. Build-first commands read it after the build.

## Resource commands

```sh
cf zones list [--name example.com]
cf dns records list --zone example.com [--type A] [--name test.example.com] [--page N --per-page N]   # one page
cf dns records create --zone <ZONE_ID> --body '{"type":"A","name":"test","content":"192.0.2.1","proxied":true}' --dry-run
cf dns records create --zone example.com --body '{...}'           # prints record with id
cf dns records delete <RECORD_ID> --zone example.com [--force]
cf schema zones create
```

- `--dry-run` prints `{ command, method, url, pathParams, query, bodyKind, body }`.
- `cf schema` prints `operationId, httpMethod, path, pathParams, queryParams, hasRequestBody, requestBodyFields`. Option names = kebab-case body fields, nested joined (`read-replication-mode`), arrays typed `string`, some lists incomplete → use `--body`.
- Output: lists = one JSON page (paging flags vary); no-data changes print nothing; raw content (R2 object, AI image) streams to stdout, redirect it; JSON always indented, colored only on TTY; failure = non-zero exit, error on stderr.
- Non-interactive (no TTY, `CI`, or CI provider detected): prompts take defaults or fail; destructive commands print `Aborted.`, exit 0 without `--force`; `cf init` needs a directory.

## cf init

- `cf init my-worker [--package-manager npm|pnpm|yarn|bun] [--no-install]` → `.cloudflare/types/index.d.ts`, `src/index.ts`, `.gitignore` (`.cloudflare/`, `.env*`), `cloudflare.config.ts`, `package.json` (dev/build/deploy/typecheck; deps `cf`, `@cloudflare/vite-plugin@beta`; no Wrangler), `tsconfig.json`, `vite.config.ts`.
- `cf init .` on an existing app: detects framework, installs `cf` + Vite plugin, adds plugin to `vite.config.ts`, writes `cloudflare.config.ts` (observability on), adds `deploy` script, adds `.wrangler`, `.dev.vars*`, `.env*` to `.gitignore`. Does NOT add `.cloudflare/`.
- Detection ≠ builds (Astro 6+ does not build during beta).

## How cf runs a project

No bundler of its own. Runs (1) the detected framework's dev/build command via your package manager (Vite: `npx vite` / `npx vite build`), else (2) the declared build tool: `@cloudflare/vite-plugin` (2.0 beta, no Wrangler dep) or Wrangler ≥4.136.0 (settings in generated `wrangler.config.ts`).

- package.json scripts are not run. Chain: `"build": "tsc -b && cf build"`.
- Framework commands get only `--mode`; `cf dev --port` fails in Vite (use `server.port`). Only Vite and Astro accept `--mode`.
- Installed build tool: `cf dev` forwards extra args; `cf build` accepts only `--mode`.

## Automatic configuration

Runs when no `cloudflare.config.ts` in cwd, from `cf dev`, `cf build`, `cf deploy` (incl. `--dry-run`), `cf previews deploy`, `cf workers versions create|triggers deploy|check`, `cf init`. Interactive: asks. CI: applies silently (package.json, lockfile, .gitignore, vite.config.ts, installs). `--prebuilt` skips it. Ignores Wrangler config → wrong config or `cloudflare.config.ts is required when --experimental-new-config is enabled.` Undo in a Wrangler project: `git status`, `git restore`, delete `cloudflare.config.ts`, `wrangler.config.ts`, `.cloudflare/`, then `cf migrate`.

## Commands

| Command | Notes |
| --- | --- |
| `cf build [--mode m]` | validates `.cloudflare/output/v0/`; no upload, no creds |
| `cf deploy` | build → validate → creds/account → upload Version → deploy. `--dry-run` (no API, no creds, still auto-configs), `--message`, `--tag`, `--secrets-file <JSON|.env>`, `--dispatch-namespace`, `--containers-rollout immediate|gradual|none`, `--worker <name>`, `--prebuilt`, `--mode` |
| `cf workers versions create` | upload only; deploy flags + `--preview-alias`. Then `cf workers deployments create --worker <name> --strategy percentage --versions '[{"version_id":"<ID>","percentage":100}]'` |
| `cf workers triggers deploy` | routes, custom domains, workers.dev, crons, queue consumers, Workflows from Build Output; `--prebuilt`, `--mode`, `--worker`, `--dry-run` |
| `cf previews deploy [name]` | `isPreview: true`; prints `{ type, version, preview_id, preview_name, preview_slug, preview_urls, deployment_id, deployment_urls }`. Name = branch (Workers Builds, `GITHUB_HEAD_REF`/`GITHUB_REF_NAME`, `CI_COMMIT_REF_NAME`, git); pass one on detached HEAD. `--mode`, `--worker`, `--prebuilt`; no `--dry-run`; needs creds. Preview output refused by deploy/versions/triggers. |
| `cf workers check [--outfile p]` | local startup profile → `worker-startup.cpuprofile`; `--prebuilt`, `--mode`, `--worker` |
| `cf workers types [--no-include-runtime] [--mode m]` | `.cloudflare/types/index.d.ts` |

### Prebuilt + mode

Build Output records `buildContext.mode` in `.cloudflare/output/v0/config.json`. `--prebuilt` must pass exactly that mode, or none if none recorded. Vite always records (default `production`); Wrangler only when `--mode` was passed to `cf build`.

```sh
cf build && cf deploy --prebuilt --mode production
cf build --mode staging && cf deploy --prebuilt --mode staging
```

## Local data

`--local` spins a short-lived runtime per command. Supported: `kv keys get|list|put|delete`, `kv bulk get|put|delete`, `kv namespaces list`, `d1 raw`, `d1 migrations list|apply`, `d1 list`, `r2 objects get|put|list|bulk-delete`, `r2 buckets list`. Else `This command has no local equivalent.` (`cf d1 query` → `cf d1 raw <ID> --sql`). State `<cf config dir>/state/v3`, shared across projects; `--persist-to <DIR>` → `<DIR>/v3`. Not the dev server's data (`.cloudflare/state/`).

`cf d1 migrations apply <DATABASE_ID> [--dir ./migrations] [--pattern ...] [--table d1_migrations] [--local]`: remote by default.

`cf dev` inside a supported coding agent prints the Local Explorer API URL (local binding data, traces, logs).

## CI

```yaml
name: Deploy
on: { pull_request: , push: { branches: [main] } }
jobs:
  deploy:
    runs-on: ubuntu-latest
    env: { CF_SEND_TELEMETRY: "false" }
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v6
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - run: npx cf build
      - run: npx cf deploy --prebuilt --mode production --dry-run
      - if: github.event_name == 'push'
        run: npx cf deploy --prebuilt --mode production
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
```

Secrets only on the deploy step, so fork PRs pass the dry run. Multi-account token without account → `More than one account available but unable to select one in non-interactive mode.` → set `CLOUDFLARE_ACCOUNT_ID` or `accountId`. Commit `cloudflare.config.ts` first.

## Errors → fixes

| Error | Fix |
| --- | --- |
| `No Cloudflare dev-server is installed in this project.` | `cf init`, or add Wrangler/Vite plugin dev dep |
| cannot resolve `@cloudflare/vite-plugin` / declared but not installed | install deps; Wrangler must be in the Worker package's own `node_modules`, not hoisted |
| `Arguments cannot currently be forwarded to the detected dev command` | framework config, or run framework command directly |
| `... does not currently support --mode` | only Vite/Astro; drop `--mode` |
| `Build Output was created with mode "production", but this command did not specify a mode` | add `--mode production` |
| `Build Output does not record which mode ... requested mode "staging"` | `cf build --mode staging` |
| `no root config found at .../.cloudflare/output/v0/config.json` | `cf build` or restore the output dir |
| `This build output is for a Preview. Run cf previews deploy --prebuilt --mode production instead.` | do that, or `cf build` |
| `We couldn't determine a Preview name from CI or Git.` | `cf previews deploy <NAME>` |
| `This command has no local equivalent.` | drop `--local`, or `cf d1 raw` |
| `cloudflare.config.ts is required when --experimental-new-config is enabled.` | `cf migrate` |
| `cf requires wrangler@4.136.0 or newer` | update Wrangler |
| `Unknown command: migrate` | `npx cf@latest migrate` or update project `cf` |
