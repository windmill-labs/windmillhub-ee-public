# Private Windmill Hub

## Setup

- Fill in the `.env` file with your desired database password, the url of your Windmill instance and your license key.
- Run `docker-compose up -d` to start the hub and database.
- Update the instance settings in the Windmill instance to point to the hub.
- Optionally, you can restrict access to the hub by setting the `API_SECRET` environment variable in the `.env` file. Make sure to set it as well in the Windmill instance settings.
- Optionally, you can set the `PUBLIC_APP_ACCESSIBLE_URL` environment variable (`APP_ACCESSIBLE_URL` in the `.env` file) to a different url than the `PUBLIC_APP_URL` (`APP_URL` in the `.env` file) if you have a different url to access your windmill instance from the user's browser.

You can enable HTTPS handling by Caddy by adding the 443 port to the Caddy service in the docker-compose.yml file.

### Required configuration on the Windmill instance for authentication

Authentication on the Hub is done through the Windmill instance. Both the Hub and the Windmill instances need to be running on the same domain for the authentication to work.
For instance, if the Windmill instance is available on `windmill.example.com`, the Hub should be accessible on a similar subdomain like `hub.example.com`. You will also need to set `COOKIE_DOMAIN` in the .env file of the **Windmill instance (server container)** to the root domain (e.g. `example.com`). After setting the `COOKIE_DOMAIN`, **make sure to logout and login again.**

### Debugging

- You can enable debug logs by setting `DEBUG_LOG=true` in the environment section of the hub service in the docker-compose.yml file.
- The hub exposes a debug page at `/debug` that displays the configuration and status of the hub.
- If you need more support, please share with us the logs and and the debug page output.

## Usage

As described above, logging in on the hub will redirect you to the Windmill instance you specified in the `.env` file.
Superadmins of the Windmill instance can approve resource types, scripts, flows, and apps on the Hub.
**Only approved content will be available to users on the Windmill instance.**

## Import scripts from the official Windmill hub

You can use the [hub cli](https://www.npmjs.com/package/@windmill-labs/hub-cli) to pull scripts locally from the official Windmill hub and push them to your private hub.

## Hub ↔ GitHub sync (approval workflows)

The hub can fire GitHub `repository_dispatch` events when content is created or
approved, so a **content repo** (the git repo that holds your `hub/` content and
that you `pull`/`push` with the hub cli) can react. Example workflows live in
[`examples/github-workflows`](examples/github-workflows) — copy the ones you need
into that repo's `.github/workflows/`. They are kept here as inert templates so
they don't run in this deployment repo. Any hub can opt in (unset = disabled).

### Pick a setup

How you wire the workflows depends on how you curate:

**A. Review every new script as a PR** — _create in UI → review PR → merge → approved._
A script created on the hub opens a PR in the content repo; merging it approves
the script on the hub. Best when you want a human gate on every submission **and**
your hub's pending asks are all things you intend to review.
Use: `open-pr-on-script-created` + `push-to-hub` + `pull-on-script-approved`
(+ optional `open-pending-prs` backstop, + optional `test-push-to-hub` preview).

**B. Approve in the UI, just sync to the repo** — _approve in UI → repo catches up._
No PR per submission. Maintainers approve in the hub UI as usual and the repo
pulls the approved content. Best when most submissions are **intentionally left
unapproved** (e.g. a public-facing hub), so a PR per submission would be noise.
Use: `pull-on-script-approved` only (+ `push-to-hub` if curators also author
content in the repo and push it).
The hub still fires `hub-script-created`, but with no listener GitHub just drops
it (204, no workflow run) — harmless, no error.

### How setup A flows

```
create script in hub UI
        │  hub fires repository_dispatch: hub-script-created
        ▼
open-pr-on-script-created.yaml ──► review PR  (+ test-push-to-hub.yaml preview, if PR_TOKEN set)
        │  reviewer reads the diff and merges
        ▼
push-to-hub.yaml ──► hub-cli push ──► script approved on the hub

# maintainer fast-track (either setup):
approve in hub UI ──► hub fires hub-script-approved ──► pull-on-script-approved.yaml ──► commit to main
```

### The workflows

- **`open-pr-on-script-created.yaml`** — triggered by `hub-script-created`. Runs
  `materialize-ask` to write the new script's files and opens/updates a review PR
  on a `hub-submission/script-<ask_id>` branch. Forward-only — only new creations,
  never the existing backlog.
- **`pull-on-script-approved.yaml`** — triggered by `hub-script-approved` (fires
  only on **UI** approvals; the CLI push approves with `?no_dispatch=true`). Pulls
  the approved content and commits to `main`; in setup A it also closes the
  matching review PR if one was open. The event-driven replacement for a polling
  "update-from-hub" cron.
- **`push-to-hub.yaml`** — runs on push to `main` touching `hub/**`. Reconciles
  the repo into the hub via `hub-cli push`, which **approves** whatever is in the
  repo — so merging a submission PR approves that script. (Omits `--prune` by
  default; see the file header for why under the "keep both approval paths" model.)
- **`test-push-to-hub.yaml`** _(optional preview)_ — dry-run `push` that comments
  on a PR what merging would do on the hub. See "The dry-run preview" below.
- **`open-pending-prs.yaml`** _(optional backstop — setup A only)_ — scheduled
  every 30 min. Runs `list-pending` and opens a PR for any unapproved ask the
  dispatch missed. **Caution:** `list-pending` returns _every_ unapproved ask not
  already in the repo, so this opens a PR for your entire unapproved backlog. Use
  it only on a curated hub where "pending == awaiting approval" — **not** on a hub
  with intentionally-unapproved asks, where it would mass-open PRs.

### The dry-run preview (optional)

By default the workflows open PRs with the built-in `GITHUB_TOKEN` and there is
**no** dry-run preview. You usually don't need one: merging a submission PR just
**approves the already-existing version** on the hub (the script was created in
the UI, so its version already exists) — a new version is pushed only if a
reviewer **edits** the file in the PR, and reviewers read the actual code in the
PR diff anyway.

To enable the preview, set `secrets.PR_TOKEN` (a PAT/GitHub App token) and add
`test-push-to-hub.yaml`. The `open-*` workflows open PRs with
`${{ secrets.PR_TOKEN || secrets.GITHUB_TOKEN }}`, so a PR opened with the PAT can
trigger `test-push-to-hub` — a PR opened with the default `GITHUB_TOKEN` can't
trigger other workflows.

### Configure the hub

Set these on the **hub** service (in `docker-compose.yml` / `.env`):

- `GITHUB_DISPATCH_REPO` — `owner/repo` of your content repo
- `GITHUB_DISPATCH_TOKEN` — token allowed to create dispatch events (fine-grained
  PAT with **Contents: write**, or a GitHub App installation token)

### Configure the content repo

In the GitHub repo holding your `hub/` content:

- `secrets.HUB_TOKEN` — a superadmin token of your Windmill instance _(required)_
- `vars.HUB_URL` — your hub URL, e.g. `https://hub.example.com` _(required; the
  examples read `vars.HUB_URL`)_
- `secrets.LICENSE_KEY` — your enterprise license key (drop it for an unlicensed
  private hub)
- `secrets.PR_TOKEN` — **optional**, only for the dry-run preview (see above).
  Without it, PRs open with `GITHUB_TOKEN`, which requires **Settings → Actions →
  General → Allow GitHub Actions to create and approve pull requests** to be
  enabled.

Requires `@windmill-labs/hub-cli` recent enough to include `list-pending` and
`materialize-ask`, and a hub that exposes `latest_version` on
`/searchData?all=true`.
