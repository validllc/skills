# cloudflare.config.ts

`@cloudflare/config` 0.20.0, open beta. Source: https://developers.cloudflare.com/cf/projects/cloudflare-config/index.md, https://developers.cloudflare.com/cf/projects/config-explorer/index.md

## Requirements

- Node ≥22.18. Bun: `cloudflare.config.ts loading is not supported on Bun`.
- package.json `"type": "module"` (else warning; `"commonjs"` fails).
- `npm i -D cf` so `cf/config` resolves.
- Nearest `cloudflare.config.ts` (walking up) is loaded by most commands. API commands read only `accountId`/`complianceRegion`, but any syntax/import/top-level error still breaks them.
- Import other config files with the `.ts` extension (`../api/cloudflare.config.ts`). Without it: `ERR_MODULE_NOT_FOUND`, and API commands in that dir break too.

## Minimum

```ts
import { defineConfig } from "cf/config";
import * as entrypoint from "./src/index.ts" with { type: "cf-worker" };

export default defineConfig({
  worker: { name: "example-worker", entrypoint, compatibilityDate: "2026-08-24" },
});
```

- `name`, `compatibilityDate` required. `entrypoint` required unless assets-only.
- `cf-worker` import attribute → typed module, bindings, exported classes. String path (`entrypoint: "src/index.ts"`) works but loses export types. `.ts` imports need `allowImportingTsExtensions`.
- No `assets.directory`. Vite: client build output (incl. `publicDir`). Wrangler bundler: `assetsDirectory` in generated `wrangler.config.ts`.

## Top level

| Field | Purpose |
| --- | --- |
| `accountId?` | default account; `CLOUDFLARE_ACCOUNT_ID` overrides |
| `complianceRegion?` | `"public"` or `"fedramp-high"`; env var overrides |
| `worker?` | the Worker (required for dev/build) |
| `containers?` | `defineContainer()[]` |

`defineConfig()`, `defineWorker()`, `defineContainer()` accept object, promise, or `(ctx) => object | promise`. `ctx = { mode: string | undefined, isPreview: boolean }`. Return one complete config per mode; nothing merges. Default export must be `defineConfig()`; the others are for reuse/import.

| Command | default `mode` |
| --- | --- |
| `cf dev` (Vite) | `development` |
| `cf build`/`cf deploy` (Vite) | `production` |
| builds via Wrangler | `undefined` |
| API commands, and account resolution in `cf deploy`/`versions create`/`triggers deploy` | `undefined` (`isPreview: false`) |

Keep `accountId` mode-independent, or pass `--mode` to every command. A mode deploys a separate Worker only when it returns a different `name`.

```ts
export default defineConfig(({ mode, isPreview }) => {
  const staging = mode === "staging";
  return {
    worker: {
      name: staging ? "example-worker-staging" : "example-worker",
      entrypoint,
      compatibilityDate: "2026-08-24",
      env: { API_ORIGIN: bindings.text(staging ? "https://staging-api.example.com" : "https://api.example.com") },
    },
  };
});
```

## Worker fields

| Field | Type |
| --- | --- |
| `name` (req) | string |
| `compatibilityDate` (req) | `yyyy-mm-dd` |
| `compatibilityFlags` | `string[]`, default `[]` |
| `entrypoint` | `string | WorkerModule` |
| `assets` | `{ htmlHandling?: "auto-trailing-slash"|"drop-trailing-slash"|"force-trailing-slash"|"none"; notFoundHandling?: "single-page-application"|"404-page"|"none"; runWorkerFirst?: string[]|boolean }` runtime only |
| `domains` | `string[]` custom domains (routes go in `triggers.fetch()`) |
| `triggers` | `Trigger[]` |
| `tailConsumers` | `{ worker: string; streaming?: boolean }[]` |
| `cache` | `{ enabled: boolean; crossVersionCache?: boolean }` |
| `placement` | `{ mode: "off"|"smart"; hint? }` or `{ mode?: "targeted"; region|host|hostname }` |
| `limits` | `{ cpuMs?; subrequests? }` standard usage model only |
| `logpush` | boolean; does not create the job |
| `observability` | `{ enabled?, headSamplingRate?, redactQueryString? (false), issues?: { enabled? }, logs?: { enabled?, headSamplingRate?, invocationLogs?, persist? (true), destinations? }, traces?: { enabled?, headSamplingRate?, persist?, destinations? } }` |
| `workersDev` | boolean, default `true` |
| `previewUrls` | boolean, default `false` |
| `unsafe` | `{ metadata?: Record<string, unknown>; capnp? }` forwarded verbatim |
| `env` | `Record<string, Binding>` |
| `exports` | `Record<string, Export>` |

## Bindings (`worker.env`)

Key = `env.<KEY>`. Use builders, never raw `type` objects. `dev?: BindingDevOptions` on most (`dev: { remote: true }` uses the remote resource in `cf dev`; default local). `cf deploy` provisions missing resources for bindings without IDs but does not write IDs back.

| Builder | Options |
| --- | --- |
| `text(value)` | literal string |
| `json(value)` | literal JSON |
| `secret()` | required secret, type `string`; replaces .dev.vars inference, warns in dev when missing |
| `d1({ id?, name?, dev? })` | |
| `kv<TKey>({ id?, dev? })` | |
| `r2({ name?, jurisdiction?, dev?: { experimentalS3Credentials? } })` | |
| `hyperdrive({ id, dev?: { connectionString? } })` | |
| `analyticsEngineDataset({ name? })` | |
| `artifacts({ namespace, dev? })` | git-compatible file storage |
| `pipeline<TRecord>({ name, dev? })` | |
| `ai({ dev? })` | Workers AI |
| `aiSearch({ name, dev? })` | |
| `aiSearchNamespace({ namespace, dev? })` | |
| `agentMemory({ namespace, dev? })` | |
| `browser({ dev? })` | |
| `images({ dev? })` | |
| `media({ dev? })` | |
| `stream({ dev? })` | |
| `vectorize({ name, dev? })` | |
| `queue<TBody>({ name?, deliveryDelay?, dev? })` | producer; consumer = `triggers.queue()` |
| `sendEmail({ destinationAddress } \| { allowedDestinationAddresses }, allowedSenderAddresses?, dev?)` | the two destination forms are exclusive |
| `assets()` | the Worker's own static assets |
| `dispatchNamespace({ namespace?, outbound?: { worker, parameters? }, dev? })` | |
| `durableObject({ worker, exportName })` | `worker` = name or `defineWorker()` import; class must be in that Worker's `exports` |
| `worker({ worker, exportName?, props?, dev? })` | service binding |
| `workerLoader()` | |
| `workflow({ name, worker, exportName })` | |
| `mtlsCertificate({ id, dev? })` | |
| `rateLimit({ namespace, simple: { limit, period: 10 \| 60 } })` | |
| `secretsStoreSecret({ storeId, secretName })` | |
| `vpcNetwork({ tunnelId } \| { networkId }, dev?)` | exclusive |
| `vpcService({ id, dev? })` | |
| `flagship({ id?, dev? })` | |
| `logfwdr({ destination })` | |
| `versionMetadata()` | |

Cross-Worker typing: pass a `defineWorker()` import as `worker` to validate `exportName` and type RPC/DO stubs. A string name validates shape only.

```ts
// api/cloudflare.config.ts
export const apiWorker = defineWorker({ name: "api-worker", entrypoint, compatibilityDate: "...",
  exports: { Admin: exports.worker(), Counter: exports.durableObject({ storage: "sqlite" }) } });
export default defineConfig({ worker: apiWorker });
// web/cloudflare.config.ts
import { apiWorker } from "../api/cloudflare.config.ts";
env: { API: bindings.worker({ worker: apiWorker, exportName: "Admin" }),
       COUNTERS: bindings.durableObject({ worker: apiWorker, exportName: "Counter" }) }
```

## Triggers (`worker.triggers`)

| Builder | Options |
| --- | --- |
| `fetch({ pattern, zone? })` | route; `zone` = name or ID, required if ambiguous |
| `queue({ name, deadLetterQueue?, maxBatchSize?, maxBatchTimeout?, maxConcurrency?, maxRetries?, retryDelay?, visibilityTimeoutMs? })` | consumer |
| `scheduled({ schedule })` | one cron per entry |
| `email({ addresses: string[] })` | literal or `*@domain` |
| `connect({ protocol: "tcp"|"udp", port, address? (127.0.0.1), idleTimeoutMs?, maxPendingBytes? })` | raw sockets; last two UDP only |

Triggers create no `env` bindings.

## Exports (`worker.exports`)

Keys must match `default` or a named class the entrypoint exports. Builders describe code; they do not create JS exports.

- `exports.worker({ cache?: { enabled } })`: `WorkerEntrypoint`, incl. `default`.
- `exports.workflow({ name, limits?: { steps? }, concurrency?: { limit? }, schedules?: string|string[], defaultRetention?: { successRetention?, errorRetention? } })`: `name` unique per account; retention = ms or `"3 days"`. Bind: `bindings.workflow({ name, worker, exportName })`.
- `exports.durableObject({...})`: see below.

### Durable Object lifecycle (replaces Wrangler `migrations`)

| `state` | Options | Meaning |
| --- | --- | --- |
| `"created"`/omitted | `storage: "sqlite"|"legacy-kv"`, `container?` | live class |
| `"deleted"` | | retire; remove bindings and class from code first |
| `"renamed"` | `renamedTo` | old key = tombstone; target must be live in same map |
| `"expecting-transfer"` | `transferFrom`, `storage`, `container?` | destination Worker, deploy first |
| `"transferred"` | `transferredTo` | source Worker, deploy second; same account |

- `sqlite` for new classes. Containers only on `sqlite` live/incoming entries.
- Do not replay applied migrations; add tombstones only for pending changes. Remove a tombstone once a deployment response marks it stale.
- Lifecycle applies on deploy, not upload. Use `cf deploy`, go to 100% before splitting traffic, no rollback past a lifecycle change.
- Live exports become `ctx.exports.<Class>` (needs `compatibilityFlags: ["enable_ctx_exports"]`); no `env` binding needed for own classes. Tombstones absent.

| | |
| --- | --- |
| `exports.Counter` | declares class, manages namespace |
| `env.COUNTERS` | namespace binding |
| `ctx.exports.Counter` | same-Worker access, no binding |

## Containers

```ts
const imageProcessor = defineContainer({ name: "image-processor", image: { dockerfile: "./Dockerfile" }, instanceType: "lite", maxInstances: 1 });
export default defineConfig({
  worker: { ...,
    exports: { ImageProcessor: exports.durableObject({ storage: "sqlite", container: imageProcessor }) },
    env: { IMAGE_PROCESSOR: bindings.durableObject({ worker: "image-worker", exportName: "ImageProcessor" }) } },
  containers: [imageProcessor],
});
```

Unique names; one Container per DO export. `cf deploy` applies Container changes (`--containers-rollout`); `versions create` only prepares images. Previews unsupported.

## Types

Vite plugin writes `.cloudflare/types/index.d.ts` on dev/build; else `cf workers types [--no-include-runtime] [--mode m]`. `cf init` adds `"typecheck": "cf workers types && tsc"`.

```json
{ "compilerOptions": { "allowImportingTsExtensions": true, "noEmit": true },
  "include": ["src", "cloudflare.config.ts", ".cloudflare/types"] }
```

`Env`: `text` → literal, `json` → literal, `d1` → `D1Database`, `queue<Job>` → `Queue<Job>`, `secret` → `string`. Use `satisfies ExportedHandler<Env>`.
