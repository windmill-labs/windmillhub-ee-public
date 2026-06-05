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

## PR-gated approval (GitHub)

Instead of approving scripts only from the hub UI, you can review them as GitHub
pull requests: **a script created on the hub opens a PR in a content repo, and
merging the PR approves it on the hub.** Example workflows are in
[`examples/github-workflows`](examples/github-workflows) — copy them into the
`.github/workflows` of the git repo that holds your `hub/` content (the repo you
`pull`/`push` with the hub cli). They are kept here as inert templates so they
don't run in this deployment repo.

### How it flows

```
create script in hub UI
        │  hub fires repository_dispatch: hub-script-created
        ▼
open-pr-on-script-created.yaml ──► review PR (test-push-to-hub.yaml comments the impact)
        │  reviewer merges
        ▼
push-to-hub.yaml ──► hub-cli push ──► script approved on the hub

# maintainer fast-track (no PR):
approve in hub UI ──► hub fires hub-script-approved ──► pull-on-script-approved.yaml ──► commit to main
```

`open-pending-prs.yaml` is a scheduled backstop that opens PRs for any creation
event the hub dropped, so the loop is resilient even if a dispatch is missed.

### Configure the hub

Set these on the **hub** service (in `docker-compose.yml` / `.env`) so it
dispatches events to your content repo (any hub can opt in — unset = disabled):

- `GITHUB_DISPATCH_REPO` — `owner/repo` of your content repo
- `GITHUB_DISPATCH_TOKEN` — token allowed to create dispatch events (fine-grained
  PAT with **Contents: write**, or a GitHub App installation token)

### Configure the content repo

In the GitHub repo holding your `hub/` content, set:

- `secrets.HUB_TOKEN` — a superadmin token of your Windmill instance
- `secrets.LICENSE_KEY` — your enterprise license key (drop it for an unlicensed
  private hub)
- `secrets.PR_TOKEN` — a PAT/GitHub App token used to **open** the review PRs.
  Required so those PRs trigger `test-push-to-hub.yaml` (PRs opened with the
  default `GITHUB_TOKEN` don't trigger other workflows)
- `vars.HUB_URL` — your hub URL (e.g. `https://hub.example.com`)

Requires `@windmill-labs/hub-cli` recent enough to include the `list-pending` and
`materialize-ask` commands, and a hub that exposes `latest_version` on
`/searchData?all=true`.
