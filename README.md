# fp-skills

Claude Code plugin shipping operational skills for **FrankenPress**
tenant site repos. Invoke skills as `/fp:<skill-name> [args]` from
inside any repo forked from
[`frankenpress/site-template`](https://github.com/frankenpress/site-template).

Skills here automate the same flows documented at
<https://docs.frankenpress.com/customizing> — they don't replace the
docs, they save you the typing.

## Install

This repo is **both a Claude Code marketplace and the plugin source**.
From any Claude Code session (project-scoped or global), run:

```text
/plugin marketplace add frankenpress/fp-skills
/plugin install fp@fp-skills
```

Verify with `/plugin list` — you should see `fp` in the output, and
`/fp:add-plugin` should be available as a slash command.

- `frankenpress/fp-skills` is the **marketplace** (the repo).
- `fp` is the **plugin name** (so skills resolve as `/fp:<skill>`).

## Skills

| Skill | Invocation | What it does |
|---|---|---|
| `add-plugin` | `/fp:add-plugin <slug>` | `composer require wpackagist-plugin/<slug>` on a `feat/add-<slug>` branch, commit `composer.json` + `composer.lock`. Stops before push / PR / activation. |

More skills land as the operational surface grows. The bar for a new
skill: it has to map to a real cross-tenant chore from
[`site-template/CLAUDE.md`](https://github.com/frankenpress/site-template/blob/main/CLAUDE.md)'s
"Common edits" or "When you ship a release" sections.

## Scope

This plugin targets the **site repo** — Bedrock layout, immutable
image, Composer-managed plugins/themes. It does **not** touch:

- Cluster state (live pods, DB, secrets) — that's the platform's job
- GitOps repos (`gitops-fp`) — Kargo owns that
- The base runtime image — that's `frankenpress/runtime` territory

If a skill would need to do any of those, it's the wrong skill for
this plugin.

## Design conventions

- **Stage, don't ship.** Skills commit on a feature branch and stop.
  Pushing, opening PRs, and activating WP-side state are user-driven.
- **Pre-flight every destructive step.** Verify working tree is clean,
  verify we're in a FrankenPress site repo, verify the input is sane
  before any `git checkout -b` or `composer require`.
- **Don't catch up unrelated changes.** A `/fp:add-plugin` PR adds
  one plugin. If Composer wants to bump other deps to satisfy
  constraints, stop and ask.
- **Mirror site-template's CLAUDE.md.** Anything the site-template
  CLAUDE.md says not to do (commit `web/wp/`, add `humanmade/s3-uploads`
  directly, edit `web/wp-config.php`, etc.) the skills must not do
  either.

## License

Apache-2.0. See [`LICENSE`](./LICENSE).
