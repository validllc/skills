# skills

Claude Code plugin marketplace. One folder per plugin under `plugins/`, each installable on its own.

| Plugin | Covers |
| --- | --- |
| `cf-cli` | Cloudflare CLI (`cf`, beta): command discovery, `cloudflare.config.ts`, dev/build/deploy, modes, Wrangler migration, CI. From https://developers.cloudflare.com/cf/ (Sep 2026), checked against cf v1.0.0-beta.12. |

## Install

```
/plugin marketplace add validllc/skills
/plugin install cf-cli@validllc
```

Local checkout: `/plugin marketplace add /path/to/skills`. Installed plugins update from this repo.

Other agents: copy `plugins/<plugin>/skills/<skill>/` into `~/.agents/skills/`, or `npx skills add validllc/skills`.

## Layout

```
.claude-plugin/marketplace.json     every plugin in this repo
plugins/<name>/
  .claude-plugin/plugin.json        name, description, version
  skills/<name>/SKILL.md            loaded on demand
  skills/<name>/references/*.md     read when needed
```

## Add a plugin

1. `plugins/<name>/.claude-plugin/plugin.json` + `skills/<name>/SKILL.md` (frontmatter `name`, `description` starting "Use when ...").
2. Entry in `.claude-plugin/marketplace.json`.
3. `claude plugin validate .`
4. Bump `version` on every change so installs update.

## Refresh cf-cli

`cf` is beta; docs and flags change. When `cf --version` moves past v1.0.0-beta.12:

```sh
curl -s https://developers.cloudflare.com/cf/llms-full.txt > /tmp/cf-docs.md   # full manual, ~10k lines
cf --help; cf deploy --help; cf migrate --help                                   # real flags beat docs
```

Diff against `plugins/cf-cli/skills/cf-cli/`, update the pinned version in `SKILL.md`, bump `version` in `plugin.json`.
