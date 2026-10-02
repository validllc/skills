---
name: cf-cli
description: Use when running the Cloudflare `cf` CLI (the `cf` or `cloudflare` command, not Wrangler) to manage Cloudflare resources from a terminal or script, when a project has cloudflare.config.ts and needs cf dev, cf build, or cf deploy, or when migrating a Wrangler project to cf
---

# Cloudflare CLI (`cf`)

Covers the public Cloudflare API (2,900+ generated commands) and Workers projects configured by `cloudflare.config.ts`. Beta. Written against cf v1.0.0-beta.12, `@cloudflare/config` 0.20.0. Confirm flags with `<command> --help` before scripting.

## cf or Wrangler?

| Project | Use |
| --- | --- |
| `cloudflare.config.ts` exists | `cf` for everything |
| `wrangler.jsonc`/`.json`/`.toml`, no `cloudflare.config.ts` | `cf` for resource/account commands only. Wrangler for dev/build/deploy. Never run `cf init .`, `cf dev`, `cf build`, `cf deploy` here (writes a wrong config or fails). `cf migrate` first. |
| Neither | `cf init <dir>` |

## Find a command

```sh
cf cli search "create a DNS record"      # local, no creds, 5 JSON matches, ONE quoted arg
cf schema dns records create             # method, path, params, body fields
cf dns records create --zone <ZONE_ID> --body '{"type":"A","name":"test","content":"192.0.2.1"}' --dry-run
cf dns records create --zone example.com --body '{...}'
```

Search queries: action + resource type only. No names, domains, IDs, tokens. `--help` only on the chosen command, never chained. `cf schema` body lists can be incomplete; pass full `--body`.

## Rules

- stdout = JSON result, stderr = messages. Pipe to `jq`. No `--json` flag.
- IDs, not names. `cf d1 query <DATABASE_ID>`, `cf dns records delete <RECORD_ID> -z example.com`. Get IDs from list filters: `cf dns records list -z example.com --name test.example.com --type A | jq -r '.[0].id'`. `-z` takes ID or domain.
- Deletes prompt. Non-interactive without `--force`/`-f`: prints `Aborted.`, exits 0, deletes nothing. Scripts must re-list to verify. `--force` is sometimes also an API param (`cf workers delete --force` removes referenced Workers). Check `--help`.
- `--dry-run`: no creds, no API calls, no name lookup (use zone IDs).
- `--local` only: `kv keys get|list|put|delete`, `kv bulk get|put|delete`, `kv namespaces list`, `d1 raw`, `d1 migrations list|apply`, `d1 list`, `r2 objects get|put|list|bulk-delete`, `r2 buckets list`. `cf d1 query` has no local form → `cf d1 raw <ID> --sql "..." --local`. State: `~/.config/cloudflare/state/v3` (macOS `~/Library/Preferences/cloudflare/state/v3`), `--persist-to <DIR>` overrides.
- Not in cf yet: live logs → `npx wrangler tail <WORKER>`; single secret → `npx wrangler secret put <NAME> --name <WORKER>` or `cf deploy --secrets-file <PATH>` (JSON or .env).
- Node ≥22.18. Bun cannot load `cloudflare.config.ts`. `"type": "module"` in package.json.
- Global `cf` runs the project's installed `cf` when present.

## Auth

Credential, first wins: `CLOUDFLARE_API_TOKEN` (env or `.env`, not `.env.local`) → `--profile <NAME>` → `cf auth activate` dir binding → default profile (`cf auth login`; `--no-browser` on SSH, `--force` re-login). No Global API Key.
Account: `CLOUDFLARE_ACCOUNT_ID` → `accountId` in config → saved choice (`.cache/cloudflare/cloudflare-account.json` under nearest `node_modules`, else `.cloudflare/cache/`) → only account. Multiple accounts + non-interactive + nothing set = failure.
Global flags: `-q`, `-z <zone>`, `--profile`, `-m <mode>`, `--local`, `--persist-to`.

## Project commands

```sh
cf dev                                   # framework dev cmd (vite) or Vite plugin/Wrangler; forwards only --mode
cf build                                 # → .cloudflare/output/v0/, no creds
cf deploy --dry-run                      # build + validate, no creds
cf deploy [--mode m] [--message t] [--tag t] [--secrets-file p] [--worker n] [--prebuilt] [--dispatch-namespace ns] [--containers-rollout immediate|gradual|none]
cf build && cf deploy --prebuilt --mode production   # --prebuilt MUST pass the recorded mode
cf build --mode staging && cf deploy --prebuilt --mode staging
cf workers versions create [--preview-alias a]       # upload only, same flags as deploy
cf workers triggers deploy               # routes, domains, crons, queue consumers, workflows
cf previews deploy [name]                # isPreview=true, needs creds, no --dry-run
cf workers types                         # .cloudflare/types/index.d.ts
cf workers check                         # startup CPU profile
cf migrate [path] [--dry-run] [--bundler vite|wrangler] [--force] [--no-install]
```

Modes replace Wrangler envs (`--mode staging`). Function-form config returns one complete config per mode, no merging. Default mode: Vite dev `development`, Vite build `production`, Wrangler builds and API commands `undefined`. Keep `accountId` mode-independent.

No `cloudflare.config.ts` → build commands run automatic configuration (edits package.json, .gitignore, vite.config.ts; silent in CI). Run `cf init .` locally, commit. Add `.cloudflare/` to `.gitignore` yourself.

CI: `CF_SEND_TELEMETRY=false`; `npx cf build`; `npx cf deploy --prebuilt --mode production --dry-run` on PRs; real deploy on main with `CLOUDFLARE_API_TOKEN` + `CLOUDFLARE_ACCOUNT_ID`.

## References

- `references/config.md`: `cloudflare.config.ts` fields, all bindings/triggers/exports builders, DO lifecycle, containers.
- `references/projects.md`: command options, prebuilt/mode rules, previews, local data, CI, errors → fixes.
- `references/wrangler-to-cf.md`: field mapping, build settings per bundler, envs → modes, `cf migrate` follow-ups.
- `references/env-vars.md`: every `CF_*`/`CLOUDFLARE_*` variable.
- Docs: https://developers.cloudflare.com/cf/llms.txt (append `index.md` to any page URL).
