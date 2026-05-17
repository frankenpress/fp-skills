---
name: add-plugin
description: Add a WordPress plugin to a FrankenPress tenant site repo by `composer require`-ing it from wpackagist on a feature branch. Triggers on "/fp:add-plugin <slug>", "add the <slug> plugin", "install a wp plugin via composer", or any request to add a WordPress plugin in a Bedrock-layout site forked from frankenpress/site-template. Stops short of pushing, opening a PR, or activating the plugin (activation is DB state and lives in `wp plugin activate`, not the image).
---

# /fp:add-plugin

Adds a WordPress plugin to the current FrankenPress site repo using
Composer + wpackagist, then commits on a feature branch. Pushing,
opening the PR, and activating the plugin in WP are the user's call —
this skill stages the change and stops.

## When to use

The user invokes `/fp:add-plugin <slug>` from inside a FrankenPress
tenant site repo (forked from `frankenpress/site-template`,
Bedrock-layout). The argument is the **wpackagist plugin slug**, which
matches the plugin's WordPress.org slug (e.g. `wordpress-seo`, not
`yoast-seo`; `wp-super-cache`, not `wpsupercache`).

If the user asks for a premium plugin or one not on wpackagist, this
skill is the wrong tool — tell them they'll need to add a private
Composer repo (vcs or composer type) to `composer.json` first.

## Pre-flight checks

Before touching anything, verify:

1. **We're in a FrankenPress site repo.** `composer.json` exists at
   the repo root AND its `repositories` array contains an entry with
   `"url": "https://wpackagist.org"`. If not, abort and tell the
   user this skill targets repos forked from `frankenpress/site-template`.
2. **The slug is plausible.** wpackagist slugs are lowercase, hyphen-
   separated, no spaces, no `wpackagist-plugin/` prefix (the skill
   adds that). If the slug looks wrong, ask before continuing — a
   typo here costs a wasted branch.
3. **Working tree is clean** on `main` (or the user's default branch).
   `git status --porcelain` should be empty. If there are unstaged
   changes, stop and ask — never stash or discard the user's work.
4. **Plugin isn't already installed.** Grep `composer.json` for
   `"wpackagist-plugin/<slug>"`. If it's already there, tell the user
   and stop — re-adding is a no-op at best, a downgrade at worst.

## Steps

1. **Sync main.**
   ```sh
   git checkout main
   git pull --ff-only
   ```
2. **Branch.**
   ```sh
   git checkout -b feat/add-<slug>
   ```
3. **Composer require.**
   ```sh
   composer require wpackagist-plugin/<slug>
   ```
   Don't pin a version unless the user asked — Composer's default
   constraint (`^<latest>`) is what the rest of the repo uses.
4. **Verify the install landed.**
   - `web/app/plugins/<slug>/` now exists (composer/installers routes
     `type:wordpress-plugin` there per the template's
     `extra.installer-paths`).
   - `composer.lock` has a new entry for `wpackagist-plugin/<slug>`.
   - No reported conflicts in stdout.
5. **Stage and commit.**
   ```sh
   git add composer.json composer.lock
   git commit -m "feat: add <slug> plugin via composer"
   ```
   Stage **only** those two files. Composer-installed plugin code
   under `web/app/plugins/<slug>/` is gitignored by the template and
   must not be force-added — that would break the immutable-image
   contract.
6. **Report.** Tell the user:
   - Branch name and commit subject (so they can verify).
   - Next steps they'll handle: `git push -u origin feat/add-<slug>`,
     `gh pr create`, and after the image rebuild + deploy, a one-time
     `wp plugin activate <slug>` against the running pod (activation
     is DB state, not image state).
   - Link them to <https://docs.frankenpress.com/customizing#add-a-plugin>
     for the full flow.

## Don't

- **Don't push or open a PR.** The user opted in to the staged-locally
  workflow; surprise pushes break per-PR review habits.
- **Don't activate the plugin.** Activation is DB state, lives outside
  the image, and requires a running cluster pod — wrong scope for an
  image-side skill.
- **Don't add the plugin without verifying it's on wpackagist.** Bad
  slugs make Composer fail noisily, leaving a dirty branch behind.
  Pre-flight check #2 catches the common cases; if Composer errors
  anyway, delete the branch (`git checkout main && git branch -D
  feat/add-<slug>`) and report the error rather than retrying blindly.
- **Don't add `humanmade/s3-uploads` directly.** It's a transitive dep
  of `frankenpress/mu-plugin`. Direct adds risk version drift —
  flagged explicitly in the site-template CLAUDE.md.
- **Don't `composer require` with `--no-update`** or otherwise skip
  the lockfile refresh. The PR is meaningless without the lockfile
  changes.
- **Don't bump unrelated deps.** If `composer require` proposes
  updating other packages to satisfy constraints, stop and ask — the
  user wants a focused PR, not a multi-package bump.

## Examples

```
User: /fp:add-plugin wordpress-seo

Skill:
  → checks composer.json has wpackagist repo ✓
  → checks main is clean ✓
  → checks yoast not already installed ✓
  → git checkout main && git pull --ff-only
  → git checkout -b feat/add-wordpress-seo
  → composer require wpackagist-plugin/wordpress-seo
  → verifies web/app/plugins/wordpress-seo/ exists
  → git add composer.json composer.lock
  → git commit -m "feat: add wordpress-seo plugin via composer"
  → reports branch + next steps to user
```

```
User: /fp:add-plugin yoast-seo

Skill:
  → "yoast-seo" isn't the wpackagist slug for Yoast SEO — that's
    "wordpress-seo". Want me to use wordpress-seo instead?
```
