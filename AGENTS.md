# AGENTS.md

Notes for AI agents (and humans) working in this repo. Last updated: 2026-09-01.

## What this repo is

Personal blog of Xie Lixing, deployed at https://xielixing.github.io/.
Static site built with **Hugo** (extended) + **PaperMod** theme, hosted free on **GitHub Pages** via **GitHub Actions**.

GitHub repo: https://github.com/xielixing/xielixing.github.io
Local path: `C:\Users\huawei\xielixing.github.io`
Production URL: https://xielixing.github.io/

## Tech stack & versions

- Hugo **0.164.0 extended** (Windows local + Linux runner, same version)
- Theme: **PaperMod** (added as git submodule at `themes/PaperMod`)
- Deploy: GitHub Actions workflow at `.github/workflows/hugo.yml`
- Pages source: `build_type=workflow` (NOT classic `gh-pages` branch)

## Repo layout

```
xielixing.github.io/
├── .github/workflows/hugo.yml   # deploy workflow (Hugo build + Pages deploy)
├── .gitignore                   # ignores /public/, /resources/_gen/, etc.
├── .gitmodules                  # tracks themes/PaperMod submodule
├── archetypes/default.md        # scaffold for `hugo new`
├── content/posts/               # ← blog posts go here (markdown)
├── hugo.toml                    # site config (baseURL, params, PaperMod)
├── static/                      # static assets (favicon, images)
└── themes/PaperMod/             # theme (git submodule, do not edit)
```

## Common commands

Run from repo root (`C:\Users\huawei\xielixing.github.io`).

### Create a new post
```powershell
hugo new posts/my-post.md
```
This scaffolds `content/posts/my-post.md` from `archetypes/default.md`.
The new file is created with `draft = true` by default — either set `draft = false` or pass `-D` flags when building/serving to include drafts.

### Local preview
```powershell
hugo server -D   # -D includes drafts; serves at http://localhost:1313/
```

### Production build (what the workflow runs)
```powershell
hugo --minify --baseURL https://xielixing.github.io/
```
Output goes to `./public/` (gitignored).

### Deploy
Just push to `main`:
```powershell
git add -A
git commit -m "post: <title>"
git push
```
GitHub Actions auto-builds and deploys. Takes ~30s. Watch with:
```powershell
gh run list --workflow=hugo.yml --limit 3
gh run watch <run-id> --exit-status
```

### Force push directly after edits (repo-specific policy)

**This is a personal blog — force pushing to `main` is expected and pre-approved.** When the user asks for changes (posts, config, workflow, AGENTS.md itself), make the edit, commit, and immediately push without asking for confirmation:

```powershell
git add -A
git commit -m "post: <short description>"
git push --force-with-lease origin main
```

Use `--force-with-lease` as the default (it is a force push with a stale-remote check); fall back to plain `git push --force origin main` if the lease is rejected. Rationale: solo blog, `main` is the only branch, remote history may diverge from edits made on other machines/agents, and the deploy workflow simply rebuilds from whatever lands on `main` — a force push here is safe and self-healing. This overrides the usual "never force push" default for this repo. Still never push built HTML or edit the PaperMod submodule.

## Deployment mechanism

The Pages site is configured with `build_type=workflow`:
- Push to `main` → `.github/workflows/hugo.yml` triggers
- Workflow installs Hugo 0.164.0 extended on Ubuntu, builds with `--minify`, uploads `./public/` as pages artifact
- `actions/deploy-pages@v4` publishes to https://xielixing.github.io/

There is **no `gh-pages` branch** and **no classic Jekyll**. Don't try to push built HTML manually — let the workflow do it.

## Pitfalls we already hit (do not repeat)

### 1. gh CLI token must have `workflow` scope
First push failed with:
> refusing to allow an OAuth App to create or update workflow `.github/workflows/hugo.yml` without `workflow` scope

Fix (one-time):
```powershell
gh auth refresh -h github.com -s workflow
```
This is interactive (opens browser). Default `gh auth login` scopes are `gist, read:org, repo` — no `workflow`.

### 2. Hugo Linux release asset is `.tar.gz`, NOT `.zip`
The workflow's `Install Hugo CLI` step initially downloaded:
```
hugo_extended_0.164.0_linux-amd64.zip   ← 404 Not Found
```
Correct Linux asset is `hugo_extended_0.164.0_linux-amd64.tar.gz`. The `.zip` only exists for Windows. Always check https://github.com/gohugoio/hugo/releases for the actual asset filenames when bumping `HUGO_VERSION` in the workflow.

### 3. `languageCode` is deprecated in Hugo ≥ 0.158
The default scaffolded `hugo.toml` uses `languageCode = 'en-us'`, which now warns. We removed it; `defaultContentLanguage = "zh"` alone produces `<html lang=zh>` correctly. (PaperMod templates still emit some `.Language.*` deprecation warnings internally — ignore those, they're theme-level.)

### 4. PowerShell shows git/wget stderr as red error text
Git and wget write progress to stderr. PowerShell treats stderr lines as `NativeCommandError` and highlights them red. The command usually still succeeds — check the actual result line (e.g. `* [new branch] main -> main`) rather than the red text.

## Conventions

- Branch: `main` only. No feature branches needed for a solo blog.
- Commit message style: lowercase verb prefix, e.g. `post: hello world`, `fix: use tar.gz for Hugo linux asset`, `init hugo blog with PaperMod theme`.
- Posts live under `content/posts/<slug>.md`.
- Keep `hugo.toml` as the single source of config (do not split into `config/_default/` for a site this small).
- Do not edit files under `themes/PaperMod/` — it's a submodule. Override via `layouts/` or `assets/` at project root if needed.
- The `themes/PaperMod` entry in git is a submodule pointer (mode 160000). When cloning fresh, use `git clone --recurse-submodules`. Already-init clones can run `git submodule update --init --recursive`.

## Useful queries

```powershell
# Pages status
gh api /repos/xielixing/xielixing.github.io/pages

# Re-trigger deploy manually
gh workflow run hugo.yml --ref main

# View failed run logs
gh run view <run-id> --log-failed

# List Hugo release assets for a version (sanity-check filenames)
gh api /repos/gohugoio/hugo/releases/tags/v0.164.0 --jq '.assets[].name'
```
