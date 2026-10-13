---
title: "Building a Tech Blog with Hugo + GitHub Pages"
date: 2026-10-13
draft: false
tags: ["blogging", "Hugo", "GitHub Pages", "GitHub Actions", "CI/CD"]
summary: "How this blog is built and shipped: Hugo, the Blowfish theme, and a GitHub Actions pipeline that deploys to GitHub Pages on every push, plus the problems I hit along the way."
---

This post documents how I built and deployed this blog end to end: the architecture, every setup step, a line-by-line walkthrough of the deployment workflow, and the problems I ran into (and how I solved them).

- **Live site**: [sagitdev.github.io](https://sagitdev.github.io)
- **Stack**: Hugo 0.167.0 (extended) · Blowfish theme · GitHub Actions · GitHub Pages
- **Environment**: Intel Mac · macOS 15.7.9

## 1. Architecture Overview

```
Write Markdown → git push → GitHub Actions runs Hugo to build pages → Deploy to GitHub Pages → Visitors see the site
```

The blog is a **static site**: every page is generated ahead of time as HTML, CSS, JS, and image files. The server just hands those files to visitors. It runs no application code and queries no database, which makes it fast, secure, and free to host.

The practice of "push code, and it builds and ships automatically" is called **CI/CD (Continuous Integration / Continuous Deployment)**.

---

## 2. Setup Steps

### 2.1 Install Hugo

Homebrew failed on my Intel Mac (see Problem 1 in Section 4), so I used Hugo's official installer instead:

```bash
# Look up the real macOS asset names for the latest release
curl -s https://api.github.com/repos/gohugoio/hugo/releases/latest | grep browser_download_url | grep darwin

# Download and install the extended edition
curl -LO https://github.com/gohugoio/hugo/releases/download/v0.167.0/hugo_extended_0.167.0_darwin-universal.pkg
sudo installer -pkg hugo_extended_0.167.0_darwin-universal.pkg -target /
hugo version   # confirm the output includes +extended
```

### 2.2 Create the site and initialize a Git repo

```bash
hugo new site blog
cd blog
git init
git symbolic-ref HEAD refs/heads/main      # older Git doesn't support git init -b main
git rev-parse --show-toplevel              # should print /Users/tank/blog
```

### 2.3 Install the theme (Git submodule)

```bash
git submodule add --depth=1 -b main https://github.com/nunocoracao/blowfish.git themes/blowfish
```

- `git submodule add`: embeds a reference to another repository. Only the referenced version is recorded; the files aren't copied into your repo's history.
- `--depth=1`: download only the latest version, not the full history.
- `-b main`: track the theme's `main` branch.

### 2.4 Configure Blowfish

Blowfish uses several files under `config/_default/` instead of a single `hugo.toml`. Copy the theme's defaults, then delete the `hugo.toml` that `hugo new site` created in the project root so the two don't conflict:

```bash
mkdir -p config/_default
cp themes/blowfish/config/_default/*.toml config/_default/
rm hugo.toml
```

| File | Purpose | Key changes |
| --- | --- | --- |
| `hugo.toml` | Core site settings | `theme = "blowfish"`, `baseURL = "https://sagitdev.github.io/"` |
| `languages.en.toml` | Title, author info, social links | `title`, `[params.author]` |
| `params.toml` | Theme feature settings | `[homepage] layout = "profile"` |
| `menus.en.toml` | Top navigation menu | Add Posts, Tags, and a GitHub link |

### 2.5 Write a post and preview locally

```bash
hugo new content posts/hello-blog.md
hugo server -D        # open http://localhost:1313
```

The block between the `---` lines at the top of a post is called **front matter** (metadata):

```yaml
---
title: "Why I Started This Blog"
date: 2026-10-12
draft: false          # must be false to be published
tags: ["blogging", "Hugo"]
summary: "..."
---
```

### 2.6 Set up automatic deployment

Two files are needed:

- `.github/workflows/deploy.yml`: the GitHub Actions workflow. **Section 3.1, "Deployment Explained," has the full file and a section-by-section walkthrough.**
- `.gitignore`: keeps build output and local files out of the repo:

```gitignore
public/
resources/_gen/
.hugo_build.lock
.DS_Store
```

### 2.7 Push to GitHub and enable Pages

1. Create a public repository named exactly **`sagitdev.github.io`**. Don't add a README or any other initial files.
2. Push:

   ```bash
   git add .
   git commit -m "Initial blog setup with Blowfish theme and first post"
   git remote add origin https://github.com/sagitdev/sagitdev.github.io.git
   git push -u origin main
   ```
3. In the repo, go to **Settings → Pages → Source** and select **GitHub Actions**.
4. Check that the run on the **Actions** page turns green, then visit the site.

---

## 3. Key Concepts

### 3.1 Deployment Explained

#### 3.1.1 An analogy

Think of deployment as **a restaurant serving a dish**:

| Restaurant | Blog deployment |
| --- | --- |
| Your recipe | Your Markdown posts and config files |
| Handing the recipe to the restaurant | `git push` sends your code to GitHub |
| The kitchen's procedure manual | `.github/workflows/deploy.yml` |
| A borrowed, empty kitchen | **Runner**: a temporary Ubuntu virtual machine from GitHub |
| The chef | **Hugo**: turns Markdown into web pages |
| The finished dish | The generated files in `public/` |
| The packed takeout box | **Artifact**: the packaged `public/` folder |
| The serving window | **GitHub Pages**: serves the pages to visitors |

You only write the recipe (the posts). Shopping, cooking, packing, and serving all happen automatically according to the manual (deploy.yml).

#### 3.1.2 What actually happens after `git push`

```
Your computer                     GitHub
─────────────                     ──────────────────────────────────────────
git push  ──────────────────────▶ ① Receives a new commit on main
                                   │
                                   ▼
                                  ② Scans the yml files in .github/workflows/
                                     deploy.yml triggers on "push to main"
                                     → match, start the workflow
                                   │
                                   ▼
                                  ③ Allocates a fresh Ubuntu VM (runner)
                                   │
                                   ▼
                                  ④ build job (runs step by step on the VM)
                                     a. Download your repo + Blowfish theme
                                     b. Install Hugo 0.167.0
                                     c. Read Pages settings, get the site URL
                                     d. Run hugo to generate public/
                                     e. Package and upload public/ (artifact)
                                   │
                                   ▼
                                  ⑤ deploy job
                                     Publish the packaged site to GitHub Pages
                                   │
                                   ▼
                                  ⑥ Site updated, VM destroyed
                                     https://sagitdev.github.io shows the new content
```

The whole process usually takes 1–2 minutes. You can watch every step live on the repo's **Actions** page.

**Key insight**: you never build or upload `public/` yourself (that's why it's in `.gitignore`). You upload only the raw ingredients; the pages are generated on GitHub's VM.

#### 3.1.3 The full deploy.yml

The file must live at `.github/workflows/deploy.yml`. GitHub looks for workflows only in that folder.

```yaml
name: Deploy Hugo to GitHub Pages
on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0
      - uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: "0.167.0"
          extended: true
      - id: pages
        uses: actions/configure-pages@v5
      - run: hugo --minify --baseURL "${{ steps.pages.outputs.base_url }}/"
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

**YAML tip**: YAML uses **indentation** (spaces, never tabs) to express nesting. A leading `-` marks a list item, and `key: value` is a setting. One wrong indent can break the whole file.

#### 3.1.4 Section-by-section walkthrough

**Section 1: Name**

```yaml
name: Deploy Hugo to GitHub Pages
```

The workflow's name, shown in the Actions run list. Pick anything that helps you recognize it.

---

**Section 2: When to run (triggers)**

```yaml
on:
  push:
    branches: [main]
  workflow_dispatch:
```

- `push: branches: [main]`: runs automatically whenever code is pushed to `main`. Pushes to other branches don't trigger it.
- `workflow_dispatch:`: allows **manual runs**. With this line, the Actions page shows a **Run workflow** button so you can redeploy without pushing.

---

**Section 3: Permissions**

```yaml
permissions:
  contents: read
  pages: write
  id-token: write
```

When a workflow runs, GitHub gives it a temporary token (`GITHUB_TOKEN`). This section defines **what that token may and may not do**, following the principle of least privilege.

| Permission | Meaning | Why it's needed |
| --- | --- | --- |
| `contents: read` | Read the repository | build downloads your posts and config |
| `pages: write` | Publish the Pages site | deploy updates the site |
| `id-token: write` | Request an OIDC identity token | deploy proves it's allowed to publish |

**What is OIDC?** Think of it as a **temporary badge**. The traditional approach is to create a long-lived secret key, store it in the repo settings, and use it at deploy time; if it ever leaks, that's dangerous. With OIDC, the workflow requests a badge on the spot that's valid only for this run and expires when the job ends. You manage no secrets, so there's nothing to leak.

Without `pages: write` or `id-token: write`, build succeeds but deploy fails with a permissions error.

---

**Section 4: Prevent overlapping deployments**

```yaml
concurrency:
  group: pages
  cancel-in-progress: false
```

- `group: pages`: only one run in the `pages` group can execute at a time.
- `cancel-in-progress: false`: when a new run arrives, don't cancel the current one; **queue** the new one instead.

Scenario: you push twice in quick succession, and the first deployment hasn't finished. With this setting, the two deployments run one after the other instead of colliding.

---

**Section 5: The build job (generate the site)**

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
```

- `jobs:`: all jobs are defined under this key. This workflow has two: `build` and `deploy`.
- `build:`: the job's name (your choice).
- `runs-on: ubuntu-latest`: run on the latest Ubuntu VM.
- `steps:`: the steps to run, in order.

`uses:` vs. `run:`:

- `uses:` calls **a prebuilt Action** (a reusable step package), in the form `owner/name@version`, e.g. `actions/checkout@v4`.
- `run:` executes **a shell command**, exactly as if you typed it in your own terminal.

`with:` passes parameters to an Action.

**Step a: Download the code**

```yaml
      - uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0
```

The VM starts out empty. This step checks out your repository onto it.

- `submodules: recursive`: **also download submodules**, i.e. the Blowfish theme. Submodules aren't downloaded by default. **Without this line, `themes/blowfish` is an empty folder and the build fails.**
- `fetch-depth: 0`: fetch the full commit history (by default, only the latest commit). With `enableGitInfo = true`, Hugo uses Git history to show each post's "last modified" date, which is only accurate with full history.

**Step b: Install Hugo**

```yaml
      - uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: "0.167.0"
          extended: true
```

Installs Hugo on the VM.

- `hugo-version: "0.167.0"`: **pin the version** to match your local machine. With `latest`, an incompatible Hugo release could suddenly break the online build while everything still works locally, which is hard to debug.
- `extended: true`: install the extended edition, which the Blowfish docs recommend. It also matches what I run locally.

**Step c: Read the Pages configuration**

```yaml
      - id: pages
        uses: actions/configure-pages@v5
```

Reads the repo's Pages settings and outputs the site's official URL (`base_url`), e.g. `https://sagitdev.github.io`.

- `id: pages`: names this step so later steps can read its outputs.

**Step d: Build the site**

```yaml
      - run: hugo --minify --baseURL "${{ steps.pages.outputs.base_url }}/"
```

This is the actual "cooking": Hugo converts Markdown into web pages and writes them to `public/`.

- `--minify`: compresses the generated HTML, CSS, and JS by stripping extra whitespace, so files are smaller and pages load faster.
- `--baseURL "..."`: tells Hugo the site's official address, used to build every link.
- `${{ steps.pages.outputs.base_url }}`: GitHub Actions **expression syntax**. Whatever is inside `${{ }}` is replaced with its real value at runtime. Here it means "take the `base_url` output from the step with id `pages` (step c)." The command ends up as:

  ```bash
  hugo --minify --baseURL "https://sagitdev.github.io/"
  ```

**Step e: Package and upload**

```yaml
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./public
```

Packages the `public/` folder and uploads it to GitHub as a **build artifact**. Why? Because build and deploy are separate jobs that may run on different VMs and can't share files directly. The build uploads the package so the deploy job can pick it up.

---

**Section 6: The deploy job (publish the site)**

```yaml
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

- `needs: build`: a **dependency**. Deploy runs only after build succeeds. If build fails, deploy is skipped and the live site keeps its previous version. Without this line, both jobs would start at once, and deploy would fail because the artifact doesn't exist yet.
- `environment: name: github-pages`: deploy to the environment named `github-pages`. You can see it, along with the deployment history, under **Settings → Environments**.
- `url: ${{ steps.deployment.outputs.page_url }}`: shows the site URL on the Actions page after deployment, so you can click straight through.
- `actions/deploy-pages@v4`: takes the artifact uploaded by build and publishes it to GitHub Pages, using the permissions and OIDC badge from Section 3.

#### 3.1.5 GitHub Actions concepts at a glance

| Concept | Meaning | In this project |
| --- | --- | --- |
| **Workflow** | A complete automated process, defined in YAML | `deploy.yml` |
| **Trigger (`on`)** | When it runs | Push to main; manual run |
| **Job** | A group of steps on the same VM | `build`, `deploy` |
| **Step** | A single action | Checkout, install Hugo, build… |
| **Action** | A prebuilt reusable step, called with `uses:` | `actions/checkout@v4`, etc. |
| **Runner** | The temporary VM that runs a job; destroyed afterward | `ubuntu-latest` |
| **Artifact** | Files passed between jobs | The packaged `public/` |
| **Environment** | A deployment target with history | `github-pages` |
| **Expression** | `${{ }}`, replaced with a real value at runtime | The site URL |

#### 3.1.6 Watching a deployment on GitHub

1. Open the repo and click the **Actions** tab to see every run.
   - 🟡 Yellow circle: running
   - ✅ Green check: succeeded
   - ❌ Red X: failed
2. Click a run. The left panel shows the `build` and `deploy` jobs.
3. Click a job to expand each step and read its logs. A failed step is marked red, and its log explains the error.
4. To redeploy manually: select "Deploy Hugo to GitHub Pages" on the left and click **Run workflow**, or open a run and click **Re-run all jobs** in the top right.

#### 3.1.7 Troubleshooting a failed deployment

First find **which step failed**, then check this table:

| Failed at | Common cause | Fix |
| --- | --- | --- |
| checkout | Wrong or inaccessible submodule URL | Check the theme URL in `.gitmodules` |
| Install Hugo | Invalid version number | Make sure `hugo-version` is a real release |
| Run hugo | Config syntax error, bad front matter, empty theme | Run `hugo` locally to reproduce and fix |
| configure-pages / deploy | Pages not enabled, or Source isn't GitHub Actions | Settings → Pages → Source → GitHub Actions |
| deploy | Missing `pages: write` or `id-token: write` | Check the `permissions` section |

**Rule of thumb**: most build errors reproduce locally. Run `hugo` (without `server`) before pushing. If it generates `public/` successfully on your machine, the online build will almost certainly succeed too.

#### 3.1.8 Two ways to deploy GitHub Pages

|  | Deploy from a branch | GitHub Actions (used here) |
| --- | --- | --- |
| How it works | GitHub reads files directly from a branch | Your own workflow builds, then publishes |
| Build tool | Only Jekyll is built in | Anything: Hugo, Astro, React… |
| Flexibility | Low | High; every step is customizable |
| Best for | Plain HTML or Jekyll sites | Any other static site generator |

This also explains Problem 3 in Section 4: while Source was still the default "Deploy from a branch," GitHub tried to build the Hugo project with Jekyll, which failed.

### 3.2 Other concepts

- **Static site generator (SSG)**: converts Markdown + templates into a complete website.
- **TOML**: Hugo's config format. `[section]` defines a group; `[[section]]` defines one item in an array.
- **`.gitignore`**: `public/` is a build output generated online by Actions, so it shouldn't be committed.
- **`baseURL`**: the site's official address; it affects every generated link.
- **The `sagitdev.github.io` repo name**: a repository named `username.github.io` becomes the root-domain site.

---

## 4. Problems Encountered and How I Solved Them

### Problem 1: Homebrew hung while installing Hugo, then failed

**Symptom**: `brew install hugo` sat at `./make.bash` for a long time. Afterward, `hugo: command not found`, and `brew list hugo` reported `No such keg`. The output included **"This is a Tier 3 configuration."**

**Cause**: Intel Macs are Tier 3 in Homebrew (no longer officially supported), so there are no prebuilt packages (bottles). Homebrew had to compile Hugo's dependency, Go, from source, which took a long time and ultimately failed.

**Fix**: skip Homebrew and use Hugo's official prebuilt installer.

**Lesson**: a Tier 3 warning means Homebrew is unreliable in that environment. Prefer official binaries.

---

### Problem 2: My home directory was secretly a Git repository (the sneakiest issue)

**Symptom**: `git init -b main` failed (my Git version was too old for `-b`), so the blog folder had no repository. Yet `git submodule add` **succeeded**.

**Cause**: outside a Git repository, `git submodule add` should fail with `not a git repository`. It succeeded because Git searches parent directories for a repo, and it found one: my home directory `/Users/tank` was itself a Git repository (probably from an accidental `git init` long ago). The theme had been registered in that repo instead. (At the time I was trying the PaperMod theme, which is why it appears below; I switched to Blowfish afterward.)

**Diagnosis**:

```bash
git rev-parse --show-toplevel    # printed /Users/tank, confirming the problem
```

**Fix**:

```bash
git rm -f themes/PaperMod                               # undo it in the home-directory repo
rm -rf "$(git rev-parse --git-dir)/modules/blog"        # clean up leftover cache
git init                                                # create a standalone repo in blog
git symbolic-ref HEAD refs/heads/main
git submodule add ...                                   # re-add the theme
git rev-parse --show-toplevel                           # should print /Users/tank/blog
```

**Lesson**: **when something succeeds that shouldn't, stop and find out why.** `git rev-parse --show-toplevel` tells you which repository you're actually in.

> 📌 If this happens to you: check whether your home directory is a repository with `ls -d ~/.git`. Before deleting it, run `git -C ~ log` and `git -C ~ status` to confirm nothing in it is worth keeping.

---

### Problem 3: A failed "pages build and deployment" run in GitHub Actions

**Symptom**: the Actions page showed two runs. The Hugo workflow succeeded, but "pages build and deployment" failed, and its log said `Build with Jekyll`.

**Cause**: when I pushed, Pages Source was still the default "Deploy from a branch," so GitHub automatically tried to build the Hugo project with Jekyll (a different static site generator), which was bound to fail.

**Fix**: set **Settings → Pages → Source** to **GitHub Actions**. From then on, only my Hugo workflow runs.

**Lesson**: enable Pages and choose the source before the first push to avoid this noise.

---

### Problem 4 (prevention): A post doesn't appear

**Cause**: posts created with `hugo new` default to `draft: true`, and drafts aren't published.

**Fix**: set `draft: false` before publishing. The `-D` in `hugo server -D` shows drafts locally, so seeing a post locally doesn't mean it will appear online.

---

### Problem 5 (knowledge): Can the repo be private?

**Answer**: on a free personal GitHub account, Pages works only with **public** repositories. Making the repo private disables the site (404). Paid plans allow private repos, but the site itself is still public.

**Alternatives**:

- Cloudflare Pages / Netlify / Vercel: free builds from private repos.
- Two repos: keep the source private, and push only the generated `public/` to a public repo.
- Simplest: keep the repo public and store drafts in a folder listed in `.gitignore`.

---

## 5. Command Cheat Sheet

```bash
# Everyday writing and publishing
hugo new content posts/post-name.md      # new post
hugo server -D                          # local preview (including drafts)
git add .
git commit -m "Add post: title"
git push                                # live in 1–2 minutes

# Themes and submodules
git submodule update --remote themes/blowfish   # update the theme (preview locally afterward)
git clone --recursive <repo-url>                # clone on a new machine, including the theme
git submodule update --init                     # fix it if you forgot --recursive

# Remove a submodule (PaperMod as the example)
git submodule deinit -f themes/PaperMod
git rm -f themes/PaperMod
rm -rf .git/modules/themes/PaperMod

# Diagnostics
hugo version                     # check version and extended
git --version
git rev-parse --show-toplevel    # which repo am I in?
ls -lh <file>                    # check a download's size
```

---

## 6. Ongoing Usage Notes

1. **Everything in a public repo is visible**, including the source of `draft: true` posts. Never commit passwords, API keys, or company code and business information.
2. **Preview locally before pushing.** If an online build fails, the previous version of the site stays live; fix the issue and push again.
3. **Site didn't update?** Check that the Actions run finished and is green, then hard-refresh the browser with `Cmd + Shift + R`.
4. **Compress images first** so pages load quickly and the repo doesn't bloat.
5. **Keep versions in sync**: after upgrading Hugo locally, update `hugo-version` in `deploy.yml` too.

## Wrapping up

Setting this up took longer than "install Hugo and push" suggests, but every detour taught me something: why prebuilt binaries beat a struggling package manager, how Git finds the repository you're in, and how a CI/CD pipeline turns a `git push` into a live site. Now publishing a post takes one command.

Thanks for reading.
