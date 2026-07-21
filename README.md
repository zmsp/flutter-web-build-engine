# Flutter Web Build Engine

A shared GitHub Actions build engine that remotely builds and deploys Flutter web apps on behalf of other repositories — without requiring any CI/CD setup in each individual project.

Supports two deployment targets:
- **GitHub Pages** via `trigger-build`
- **Cloudflare Pages** via `trigger-cf-deploy`

---

## How It Works

Source repositories send a `repository_dispatch` event to this engine repo. The engine checks out the source, builds the Flutter web app, deploys it to the specified target, and posts commit statuses back to the source repo.

```
Your App Repo  ──(repository_dispatch)──▶  Build Engine  ──▶  GitHub Pages
                                                          └──▶  Cloudflare Pages
```

---

## Setup

### 1. Required Repository Secrets

Configure these in **this engine repo's** Settings → Secrets and variables → Actions:

| Secret | Description |
|---|---|
| `GH_PAT` | GitHub Personal Access Token with `repo` scope — used to check out private source repos and post commit statuses |
| `CLOUDFLARE_API_TOKEN` | Cloudflare API token with **Cloudflare Pages: Edit** permission |
| `CLOUDFLARE_ACCOUNT_ID` | Your Cloudflare account ID (found in the Cloudflare dashboard sidebar) |

> **Security:** `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` are immediately masked with `::add-mask::` at the very first step of the Cloudflare job. They are never echoed or visible in any log line or UI summary. They are only passed as env vars scoped to the single wrangler deploy step.

---

### 2. Allow Source Repos to Dispatch

Each source repo needs a secret (e.g. `BUILD_ENGINE_PAT`) containing a PAT with `repo` scope on **this engine repo**, so it can POST a `repository_dispatch` event here.

---

### 3. Cloudflare Pages Project (CF deploys only)

Your Cloudflare Pages project must already exist before triggering a deployment. Create it once:

```bash
npx wrangler pages project create <project-name>
```

---

## Triggering a Build

Use the [`peter-evans/repository-dispatch`](https://github.com/peter-evans/repository-dispatch) action in your source repo's workflow:

```yaml
- name: Trigger Remote Build
  uses: peter-evans/repository-dispatch@v3
  with:
    token: ${{ secrets.GH_PAT }}   # PAT with repo scope on the engine repo
    repository: zmsp/flutter-web-build-engine
    event-type: trigger-build      # or trigger-cf-deploy
    client-payload: |
      { ... }
```

> **Note:** `secrets.GH_PAT` here must have `repo` scope on the **engine repo** so it can POST the dispatch event. The same PAT (or a separate one stored in the engine repo) is also used to check out your source repo and push to the deploy target.

---

## Dispatch Types & Payload Reference

### `trigger-build` — Deploy to GitHub Pages

```jsonc
{
  "event_type": "trigger-build",
  "client_payload": {
    // Required
    "source_repo": "owner/source-repo",   // repo to build from
    "sha":         "abc1234",             // commit SHA (for status checks)
    "ref":         "main",               // branch/tag/SHA to checkout
    "target_repo": "owner/target-repo",  // repo hosting gh-pages

    // Optional
    "sub_url":    "my-app",             // sub-path in gh-pages (e.g. /target-repo/my-app/)
    "base_href":  "/my-app/",           // overrides Flutter --base-href (highest priority)
    "site_url":   "https://example.com" // URL shown in commit status on success
  }
}
```

**`base_href` resolution priority:**
1. `base_href` payload field (explicit override)
2. `/<repo-name>/<sub_url>/` (when `sub_url` is set)
3. `/<repo-name>/` (default GitHub Pages path)

**Commit status context:** `Remote Build Engine`

---

### `trigger-cf-deploy` — Deploy to Cloudflare Pages

```jsonc
{
  "event_type": "trigger-cf-deploy",
  "client_payload": {
    // Required
    "source_repo":   "owner/source-repo",  // repo to build from
    "sha":           "abc1234",            // commit SHA (for status checks)
    "ref":           "main",              // branch/tag/SHA to checkout
    "project_name":  "my-cf-project",     // Cloudflare Pages project name

    // Optional
    "branch":     "main",                 // branch label shown in Cloudflare dashboard
                                          // defaults to the engine's ref_name
    "base_href":  "/",                    // Flutter --base-href (defaults to "/")
    "site_url":   "https://example.com"   // URL shown in commit status on success
                                          // defaults to https://<project_name>.pages.dev
  }
}
```

**Commit status context:** `Remote Build Engine / Cloudflare`

---

## Example: Full Source Repo Workflow

### GitHub Pages deploy

```yaml
name: Trigger Remote Build Server

on:
  push:
    branches:
      - master
  workflow_dispatch:

jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - name: Repository Dispatch
        uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.GH_PAT }}
          repository: zmsp/flutter-web-build-engine
          event-type: trigger-build
          client-payload: |
            {
              "source_repo": "${{ github.repository }}",
              "target_repo": "zmsp/YOUR_PAGES_REPO",
              "site_url":   "https://your-custom-domain.com",
              "base_href":  "/",
              "ref":        "${{ github.ref_name }}",
              "sha":        "${{ github.sha }}"
            }
```

> **Tip:** Use `github.ref_name` (e.g. `master`) rather than `github.ref` (e.g. `refs/heads/master`) — both work with `actions/checkout`, but `ref_name` is cleaner.

---

### Cloudflare Pages deploy

```yaml
name: Trigger Cloudflare Deploy

on:
  push:
    branches:
      - master
  workflow_dispatch:

jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - name: Repository Dispatch
        uses: peter-evans/repository-dispatch@v3
        with:
          token: ${{ secrets.GH_PAT }}
          repository: zmsp/flutter-web-build-engine
          event-type: trigger-cf-deploy
          client-payload: |
            {
              "source_repo":  "${{ github.repository }}",
              "project_name": "your-cf-project-name",
              "branch":       "${{ github.ref_name }}",
              "base_href":    "/",
              "site_url":     "https://your-custom-domain.com",
              "ref":          "${{ github.ref_name }}",
              "sha":          "${{ github.sha }}"
            }
```

---

## Utility Workflows

### `clear-cache.yml` — Manual Cache Clearing

Manually purge GitHub Actions caches in this repo via **Actions → Manual Clear Cache → Run workflow**.

| Input | Description |
|---|---|
| `key_prefix` | Filter caches by prefix (e.g. `pub-`, `build-`, `build-cf-`). Leave empty to clear all. |

---

## Cache Key Structure

| Prefix | Used by |
|---|---|
| `pub-<os>-<repo>-<hash>` | Flutter pub cache (both jobs) |
| `build-<os>-<repo>-<hash>-<sha>` | Build artifacts (GitHub Pages job) |
| `build-cf-<os>-<repo>-<hash>-<sha>` | Build artifacts (Cloudflare job) |

---

## Commit Status Checks

Both jobs post GitHub commit statuses back to the source repo:

| State | When |
|---|---|
| `pending` | Job starts |
| `success` | Build + deploy succeeded — links to the deployed URL |
| `failure` | Build or deploy failed — links to the Actions run log |
| `error` | Job was cancelled or ended unexpectedly |
