# Environment variables

Configure the CLI process, not a deployed Worker. Source: https://developers.cloudflare.com/cf/environment-variables/index.md

| Variable | Default | Effect |
| --- | --- | --- |
| `CLOUDFLARE_API_TOKEN` | unset (OAuth profile) | Auth for API commands; beats every profile. Blocks `cf auth create/activate/deactivate/delete`. |
| `CLOUDFLARE_ACCOUNT_ID` | unset | Account; overrides `accountId` in config. |
| `CLOUDFLARE_ZONE_ID` | unset | Default zone ID or domain; `-z` wins. |
| `CLOUDFLARE_COMPLIANCE_REGION` | `public` | `public`, `fedramp_high`, `fedramp-high`; overrides config. |
| `CLOUDFLARE_API_BASE_URL` | region endpoint | Overrides API endpoint. Carries credentials. |
| `CLOUDFLARE_ACCESS_CLIENT_ID` / `_SECRET` | unset | Access service token; set both. |
| `CLOUDFLARE_REGISTRY_PATH` | `registry/` in cf config dir | Local dev registry shared with Wrangler/Miniflare. |
| `CI` | unset (also auto-detected) | `true` → no prompts; defaults used, required answers fail. |
| `CF_QUIET` | unset | `1` → no progress animation (`-q`). Results still print. |
| `CF_NO_OSC_PROGRESS` | unset | `1` → no tab-title/taskbar progress. |
| `CF_FORCE_OSC_PROGRESS` | unset | `1` → terminal progress on unrecognized terminals. |
| `CF_SEND_TELEMETRY` | saved pref | `true`/`1` on, `false`/`0` off. Beats `WRANGLER_SEND_METRICS`, loses to `DO_NOT_TRACK`. Also `cf cli telemetry`. |
| `DO_NOT_TRACK` | unset | `1` → telemetry off, always. |
| `WRANGLER_SEND_METRICS` | unset | Fallback when `CF_SEND_TELEMETRY` unset. |
| `DEBUG` | unset | nonempty → stack traces, delegated command details. |
| `FORCE_COLOR` | unset | non-`0` → color without TTY; `NO_COLOR` wins. |
| `NO_COLOR` | unset | any → no color, no progress animation. |
| `TUNNEL_MANAGEMENT_TOKEN` | unset | Token for `cf tunnels tail`; omit tunnel ID when set. |

Read from project `.env` (not `.env.local`): `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_ZONE_ID`, `CLOUDFLARE_COMPLIANCE_REGION`, `CLOUDFLARE_ACCESS_CLIENT_ID`, `CLOUDFLARE_ACCESS_CLIENT_SECRET`. Everything else: process env only.
