# Wrangler → cf

Source: https://developers.cloudflare.com/cf/wrangler/index.md, /cf/wrangler/migrate/index.md, /cf/wrangler/reference/index.md

## Differences

| | Wrangler | cf |
| --- | --- | --- |
| Coverage | Workers + subset | full public API |
| Sign-in | `wrangler login` | `cf auth login`, separate credentials |
| Config | `wrangler.jsonc`/`.toml` | `cloudflare.config.ts` |
| Environments | `env` blocks, `--env` | modes, `--mode` |
| Build | Wrangler bundler | Wrangler bundler or Vite plugin |
| Artifact | internal | `.cloudflare/output/v0/` |
| Output | tables, some `--json` | JSON |
| Identifiers | names | API IDs (`cf d1 query <ID>` vs `wrangler d1 execute my-db`) |
| Local/remote | some default local | remote; `--local` only for supported KV/D1/R2 |

Unchanged: Workers, bindings, compat dates, versions, deployments. `cf migrate` keeps name, bindings, IDs, routes, triggers. `.dev.vars` still loads in `cf dev`.

Coexist: resource/account commands work in unmigrated Wrangler projects (set `CLOUDFLARE_ACCOUNT_ID`; Wrangler config is not read). Keep `wrangler dev`/`deploy` until migrated. Both read `CLOUDFLARE_API_TOKEN`/`CLOUDFLARE_ACCOUNT_ID`.

Still Wrangler: `npx wrangler tail <WORKER>`; `npx wrangler secret put <NAME> --name <WORKER>` (or `--secrets-file` on `cf deploy`/`versions create`). Wrangler ignores `cloudflare.config.ts`; keep its config while you still run it.

Discovery: `cf cli search "create a D1 database"` → `cf schema d1 create` → `cf d1 --help`. Versions/deployments: `cf workers versions`, `cf workers deployments`.

## Migrate

Prereqs: clean worktree incl. untracked (ignored files OK); deps installed; Wrangler ≥4.136.0 in the Worker package for the Wrangler bundler (≥4.100.0 to generate `wrangler.config.ts`); never run `cf dev/build/deploy` first.

```sh
cf migrate --dry-run                       # or: cf migrate packages/api/wrangler.jsonc --dry-run
cf migrate [--bundler vite|wrangler] [--force] [--no-install]
git status && git diff
```

- Finds exactly one `wrangler.json|jsonc|toml` in cwd or takes a path; writes beside it.
- Bundler: Vite if `@cloudflare/vite-plugin` declared, else Wrangler. Switching to Vite: `npm i -D vite @cloudflare/vite-plugin@beta`, `vite.config.ts` with `plugins: [cloudflare()]`. Beta plugin reads `cloudflare.config.ts`, has no `configPath` (remove it).
- Writes `cloudflare.config.ts`, `wrangler.config.ts` (Wrangler bundler only), package.json + lockfile (`cf` dev dep). Leaves Wrangler config, scripts, vite.config.ts, .gitignore, tsconfig, source untouched. Never overwrites; stops if `cloudflare.config.ts` exists (`--force` bypasses only the clean-worktree check). Installs in the package beside the config, not workspace root.
- Exit 1 while `[required]` items remain (dry run too). `[info]` = no action. Required items → `TODO(@cloudflare)` comments + top-level `throw new Error("Migration incomplete...")` blocking dev/build/deploy until deleted.

### Follow-ups

| Item | Fix |
| --- | --- |
| DO bindings (always) | review `bindings.durableObject({ worker, exportName })` |
| `migrations` (always) | not converted. `exports: { Counter: exports.durableObject({ storage: "sqlite" }) }` per live class; `"sqlite"` for `new_sqlite_classes`, `"legacy-kv"` for `new_classes`. Do not replay applied renames/deletes. |
| Environments | each `env.<NAME>` → `case "<NAME>"` in `switch (ctx.mode)`, each a complete config (top-level bindings/vars/secrets NOT inherited). Unnamed env → `<name>-<env>`. `--env X` → `--mode X`. Vite bundler: rename an env called `production`/`development` (auto-selected). |
| Build settings | Wrangler bundler: moved to `wrangler.config.ts`. Vite: required item, move by hand (table below). Custom `build` cmd: `npm run build:css && cf build`. |
| `upload_source_maps` (Vite) | `environments.ssr.build.sourcemap: true` in vite.config.ts |
| D1 `migrations_dir|pattern|table` | `cf d1 migrations apply <ID> --dir --pattern --table` (remote default) |
| `preview_id`, `preview_bucket_name`, `preview_database_id` | none; dev uses local unless `dev: { remote: true }` |
| route with `zone_id` | copied to `triggers.fetch({ zone })`; confirm |
| legacy service env binding | rewritten `<SERVICE>-<ENV>`; confirm |
| `site` | unsupported; Static Assets |
| `previews` | `ctx.isPreview` branch; review |
| missing `name`/`compatibility_date` | placeholders `"TODO"`/`"YYYY-MM-DD"` |
| unknown fields (`keep_vars`) | remove |
| Workflows | `exports.workflow({ name })` + `bindings.workflow({ name, worker, exportName })` |
| Containers | `defineContainer()` in `containers` + `exports.durableObject({ storage: "sqlite", container })` |
| Secrets | `secrets.required` → `bindings.secret()`; declare others too. Upload via `--secrets-file`. migrate never reads `.dev.vars`/`.env`. |
| Installing cf | with `--no-install`, failed install, or no package.json beside config: `npm i -D cf` |

### Finish

1. Scripts: `"dev": "cf dev", "build": "cf build", "deploy": "cf deploy"` (chain extras: `tsc -b && cf build`).
2. `.cloudflare/` in `.gitignore`.
3. `"type": "module"`.
4. Types: Vite writes on dev/build; Wrangler bundler: `types: { generate: true }` in `wrangler.config.ts` or `cf workers types`. tsconfig `include: ["src", "cloudflare.config.ts", ".cloudflare/types"]`.
5. Optional: `import * as entrypoint from "./src/index.ts" with { type: "cf-worker" }` (needs `allowImportingTsExtensions`).
6. `cf dev`; `cf build && cf deploy --dry-run` (per mode too). Then `cf auth login && cf deploy`.
7. Delete Wrangler config once scripts/CI use cf and no Wrangler commands remain.

## Field mapping

| Wrangler | cloudflare.config.ts |
| --- | --- |
| `name` | `worker.name` |
| `main` | `worker.entrypoint` |
| `compatibility_date` / `_flags` | `worker.compatibilityDate` / `compatibilityFlags` |
| `account_id` | `accountId` |
| `compliance_region` (`fedramp_high`) | `complianceRegion` (`"fedramp-high"`) |
| `vars` | `bindings.text()` strings, `bindings.json()` else |
| `secrets.required` | `bindings.secret()` |
| KV/D1/R2/Hyperdrive/... | matching `bindings.*` (preview fields = required item) |
| `services` | `bindings.worker({ worker })` |
| `queues.producers` / `consumers` | `bindings.queue({ name })` / `triggers.queue({ name, maxBatchSize, ... })` camelCase |
| Hyperdrive `localConnectionString` | `dev: { connectionString }` |
| `remote: true` | `dev: { remote: true }` |
| `routes` (`zone_name`/`zone_id`) | `triggers.fetch({ pattern, zone })` |
| `routes` `custom_domain: true` | `worker.domains` |
| `triggers.crons` | `triggers.scheduled({ schedule })` each |
| `durable_objects.bindings` | `bindings.durableObject({ worker, exportName })` |
| `migrations` | `worker.exports` (manual) |
| `workflows` | `bindings.workflow()` + `exports.workflow()` (manual) |
| `containers` | `defineContainer()` (manual) |
| `assets.binding` | `bindings.assets()` |
| `assets.html_handling`/`not_found_handling`/`run_worker_first` | `worker.assets.*` camelCase |
| `assets.directory` | build settings |
| `observability`/`limits`/`placement` | `worker.*` camelCase (`cpuMs`) |
| `tail_consumers` / `streaming_tail_consumers` | `worker.tailConsumers` (`streaming: true`) |
| `workers_dev` / `preview_urls` | `worker.workersDev` / `previewUrls` |
| `env.<NAME>` | `switch (ctx.mode)` case |
| `previews` | `ctx.isPreview` branch |
| `site` | unsupported |
| D1 `migrations_*` | `cf d1 migrations apply` flags |
| `build`, `minify`, `alias`, ... | build settings |

## Build settings

| Wrangler | Vite `vite.config.ts` | Wrangler `wrangler.config.ts` (`defineWranglerConfig` from `wrangler/experimental-config`) |
| --- | --- | --- |
| `alias` | `resolve.alias` | `alias` |
| `define` | `define` | `define` |
| `minify` | `build.minify` | `minify` |
| `upload_source_maps` | `build.sourcemap` in Worker env (`ssr`) | `uploadSourceMaps` |
| `assets.directory` | `publicDir` | `assetsDirectory` |
| `build` | run before `cf build` | `build` camelCase |
| `dev` | `server` | `dev` camelCase |
| `rules`, `tsconfig`, `no_bundle`, `find_additional_modules`, `base_dir`, `preserve_file_names` | not used | camelCase |

migrate also sets `types: { generate: false }` in `wrangler.config.ts`.

## Errors → fixes

| Error | Fix |
| --- | --- |
| `No Wrangler config found` / `Multiple Wrangler configs found` | right dir, or pass the path |
| `Git worktree is not clean` | commit/stash (incl. untracked) or `--force` |
| `Cannot migrate because .../cloudflare.config.ts already exists` | finish it; if dev/build/deploy made it, undo first |
| `Generating wrangler.config.ts requires wrangler 4.100.0 or newer` | install/update Wrangler locally |
| `cf requires wrangler@4.136.0 or newer` | update Wrangler |
| `Migration incomplete. Resolve every cf migrate TODO` | resolve, delete the `throw` |
| `cloudflare.config.ts is required when --experimental-new-config is enabled.` | `cf migrate` |
| `wrangler is declared in .../package.json but is not installed.` | install in the Worker package (not hoisted) |
| `Unknown command: migrate` | `npx cf@latest migrate` |
