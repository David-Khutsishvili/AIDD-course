# Plan 1: Ship Your Site

**Lecture 1 exercise** (see `Todo.jpeg`):

1. Create an empty GitHub repo and clone it
2. Inside it, create the app: `npx create-next-app@latest .`
3. Commit and push
4. Import the repo in Vercel and deploy
5. Prove the pipeline: branch → small change → PR → preview URL → merge → production updates
6. Post your live URL in the group

**Before you start:** Node.js, Git, GitHub and Vercel accounts, and `git config` name + email.

> Course rule: use the AI agent the whole way, and have it explain each command before you run it.

**Added to the exercise:** between Steps 4 and 5 we build a **CI quality gate** (Step 4.5): a GitHub Actions workflow that runs lint, type-check, format check and tests on every PR, plus a connected testing platform.

---

## Progress log

> Updated at the end of every session so you can continue on any PC: `git pull`, then read this section.

| Date | PC | What happened |
|---|---|---|
| 2026-11-22 | Main PC | Environment check, Step 0 (git config fixes). |
| 2026-11-25 | Main PC | Steps 1–2 done. Step 3: scaffold + `CLAUDE.md` commits pushed to `main`. |
| 2026-11-25 | Second PC | Cloned the repo. `npm ci`, `npm run lint` and `npm run build` all pass. Moved this plan into the repo (`docs/plan1.md`). Added Step 4.5 (CI). Step 4 (Vercel) in progress. |

**Current status:**
- ✅ Steps 0–2 done.
- 🟡 Step 3: commits are pushed, but the **`protect-main` ruleset and the squash-only merge settings are not set yet**. GitHub reports no rules on `main`. Do this on the main PC (it has `gh` installed).
- 🟡 Step 4: in progress on the second PC.
- ⬜ Step 4.5 (CI), Step 5 and Step 6: not started.

**Naming note:** this plan was written before the repo existed, and its examples use `code/` and `ai-course-web`. The real names are:
- GitHub repo: `David-Khutsishvili/AIDD-course`
- On the second PC: `C:\David\Projects\AIDD-course\AIDD-course\`

**Where this plan lives:** it was at the course-folder root (`plan1.md`), outside the repo, so it didn't sync between PCs. It's now committed at `docs/plan1.md`. On the main PC, once this branch is merged and pulled, delete the old copy in the course folder so you don't edit a stale one.

---

## Environment check (2026-11-22, main PC)

- ✅ Node v22.18, npm 10.9, git 2.49 and the GitHub CLI (`gh`) are installed. `gh` is logged in as `David-Khutsishvili`.
- ⚠️ **Typo in git config:** `use.name=David Khutsishvili`, so `user.name` isn't set. Commits would fail or show the wrong name.
- ⚠️ The default branch name was `master`. GitHub and Vercel use `main`.
- ⚠️ Commit email is `vibendavid@gmail.com`. Check that it's linked to the GitHub account.
- `code/` contains only `CLAUDE.md`, and the course folder is not a git repo. That's the correct starting point.

## Environment check (2026-11-25, second PC)

- ✅ Node **v24.19**, npm **11.17** and git are installed. Git `user.name` = `David Khutsishvili`, and `user.email` = `davidhuzt@gmail.com`, which matches the existing commits.
- ⚠️ **The GitHub CLI (`gh`) is not installed.** The `gh` commands in this plan only work on the main PC. On this PC, use the github.com web UI, or install it with `winget install GitHub.cli` and then run `gh auth login`.
- ⚠️ `init.defaultBranch` falls back to `master` (from Git's system config). It doesn't matter for this repo, which is already on `main`. To fix it: `git config --global init.defaultBranch main` and `git config --global fetch.prune true`.
- ⚠️ **Node versions differ between PCs** (v22 on the main PC, v24 on this one). That's fine for now. In Step 4.5 we pin one Node version for CI and Vercel so "works on my machine" means the same thing everywhere.
- Use `npm ci` (not `npm install`) after cloning or pulling. It installs exactly what's in `package-lock.json` and never rewrites it, so different npm versions on the two PCs don't create noisy lockfile diffs.

**Folder layout.** `code/` will be its own git repo, the one that goes to GitHub and Vercel. The outer course folder stays a plain folder, and repos are never nested.

---

## Step 0: Fix the prerequisites

```powershell
git config --global --unset use.name
git config --global user.name "David Khutsishvili"
git config --global init.defaultBranch main
git config --global fetch.prune true
```

**Why:** every commit permanently records an author name and email. If you fix them later, the old commits still carry the wrong values.
- `init.defaultBranch main` keeps your local branch name the same as GitHub's.
- `fetch.prune` removes your local copies of branches that were deleted on GitHub. That keeps `git branch -a` clean once you're merging PRs.

**Line endings:** you may see "LF will be replaced by CRLF" warnings. They're harmless on Windows. Git converts line endings because of `core.autocrlf=true`.

### Check and change the commit email

First, check which email git uses:
```powershell
git config --global user.email          # prints the current global email
```

Then open **GitHub → Settings → Emails** and check that this address is listed and verified. GitHub links a commit to your profile *only* by its email. If the email isn't on your account, commits show a grey "unknown" author, and Vercel may refuse to deploy commits from authors it can't match to your account.

**To change it (for all repos):**
```powershell
git config --global user.email "your-email@example.com"
git config --global user.email          # verify the change
```

**To change it for one repo only** (run inside that repo, without `--global`):
```powershell
git config user.email "other-email@example.com"
```
How it works: git reads config in layers: system → global (`C:\Users\david\.gitconfig`) → repo (`.git/config`). The most specific layer wins. A repo-level email therefore overrides the global one in that repo only. This is useful when, for example, work and personal projects use different emails.

**Privacy option (recommended for a public repo):** commit emails are public in a public repo's history. GitHub gives you a private "noreply" address that still links commits to your profile:
1. GitHub → Settings → Emails → enable **"Keep my email addresses private"**. You can optionally also enable "Block command line pushes that expose my email".
2. Get your noreply address:
   ```powershell
   gh api user --jq '\"\(.id)+\(.login)@users.noreply.github.com\"'
   ```
   (In Git Bash, use `gh api user --jq '"\(.id)+\(.login)@users.noreply.github.com"'`.)
3. Set it: `git config --global user.email "<that-address>"`

**If you already made a commit with the wrong email:**
- Not pushed yet: `git commit --amend --reset-author --no-edit` rewrites the last commit with your current name and email.
- Already pushed to `main`: leave it. Rewriting pushed history needs a force push, which the `main` ruleset blocks, and that's on purpose.

---

## Step 1: Create an empty GitHub repo and clone it

On the web, go to github.com/new:
- Name it `ai-course-web` (or your own name; then update `code/CLAUDE.md`).
- Make it **Public**. On the free plan, GitHub only enforces branch protection on public repos, which Step 3 needs.
- **Don't** tick README, .gitignore or license.

Or from the CLI: `gh repo create ai-course-web --public`

`git clone` only works into an **empty** folder, so first move `code/CLAUDE.md` aside for a moment. It comes back in Step 3.
```powershell
cd X:\Studis\year-6\AI_Driven_Development_course
mv code/CLAUDE.md CLAUDE.code.md
git clone https://github.com/David-Khutsishvili/ai-course-web.git code
```

**Why empty:** `create-next-app` won't scaffold into a folder that has files that clash with its own (this includes our `CLAUDE.md`), and it brings its own `.gitignore` and README. An empty repo also means your first commit is the real project, not a leftover README.

**How cloning works:** `git clone` creates the `.git` folder (the whole history database). It also registers the GitHub URL as a remote called `origin`, so `push` and `pull` know where to go. The warning *"You appear to have cloned an empty repository"* is expected.

Your `gh` login uses HTTPS and Windows' credential manager, so you don't need to set up SSH. The course notebook's SSH guide is another way to log in to GitHub, not a required step.

---

## Step 2: Create the Next.js app inside the repo

```powershell
cd code
npx create-next-app@latest .
```

What each part means:
- `npx` downloads and runs a package without installing it globally.
- `@latest` gets the newest template.
- `.` means "in this folder" rather than a new subfolder.

Recommended answers to the prompts (the exact questions vary by version):

| Prompt | Pick | Why |
|---|---|---|
| TypeScript | Yes | Catches type errors before runtime. This is the production standard. |
| Linter (ESLint) | Yes | Automatic code-quality checks. CI will run it later. |
| Tailwind CSS | Yes | The most common styling tool with Next.js. |
| `src/` directory | Yes | Keeps app code apart from config files. |
| App Router | Yes | The modern Next.js routing system. |
| Turbopack | Yes | The faster dev server. |
| Import alias | Keep `@/*` | Lets you write `@/components/x` instead of `../../../components/x`. |

Test it locally:
```powershell
npm run dev        # open http://localhost:3000, then Ctrl+C to stop
git status
```

**How it works:** `npm run dev` runs a local development server that reloads when you save a file. `git status` lists the new files as "untracked". You won't see `node_modules/` or `.next/` because the generated `.gitignore` excludes them:
- `node_modules/` holds the dependencies. They get reinstalled from `package-lock.json`.
- `.next/` holds build output. It gets regenerated.

Secrets (`.env*`) are ignored too. **Do commit `package-lock.json`.** It pins exact dependency versions, so your machine and Vercel install the same thing.

---

## Step 3: First commits, push, then lock `main`

```powershell
git add .
git status                                   # review exactly what's staged
git commit -m "chore: scaffold Next.js app with create-next-app"
```

Now bring back the CLAUDE.md as its own commit (one logical change per commit):
```powershell
ls                                           # did create-next-app generate its own CLAUDE.md / AGENTS.md?
mv ../CLAUDE.code.md CLAUDE.md
git add CLAUDE.md
git commit -m "docs: add CLAUDE.md project guide for AI agents"
```

> **If create-next-app generated its own files:** newer versions may add an `AGENTS.md` (Next.js docs pointers for AI agents) and/or a `CLAUDE.md`.
> - If there's a generated `CLAUDE.md`, the `mv` above fails in PowerShell. Read the generated file, delete it, then run the `mv` again.
> - If there's an `AGENTS.md`, keep it and add a line `@AGENTS.md` near the top of our `CLAUDE.md`. That syntax tells Claude Code to load the other file too.

Then push:
```powershell
git branch -M main                           # make sure the branch is named main
git push -u origin main
```

**Why these go straight to `main`:** a pull request needs a branch to merge *into*. Until `main` has commits, there's nothing to branch from or open a PR against. These bootstrap commits are the only exception. You close the door right after.

**How it works:**
- `add` stages changes. The staging area lets you choose what goes into a commit.
- `commit` saves a snapshot to your local history.
- `push -u` uploads it and sets `origin/main` as the upstream, so later a plain `git push` or `git pull` knows where to go.
- The prefixes follow **Conventional Commits** (`feat:`, `fix:`, `chore:`, `docs:`, `test:`). They make history easy to read and can be automated later.

### Enforce "never commit to main"

1. GitHub repo → **Settings → Rules → Rulesets → New branch ruleset**:
   - Name: `protect-main`
   - Enforcement: **Active**
   - Target: **Include default branch**
   - Rules: ✅ Restrict deletions · ✅ Require a pull request before merging (required approvals: **0**) · ✅ Block force pushes
2. Set the merge strategy:
   ```powershell
   gh repo edit --enable-squash-merge --enable-merge-commit=false --enable-rebase-merge=false --delete-branch-on-merge
   ```
   **Without `gh`** (web UI): open repo → **Settings → General → Pull Requests**. Tick only **"Allow squash merging"**, untick "Allow merge commits" and "Allow rebase merging", and tick **"Automatically delete head branches"**.

**Why:**
- **Approvals = 0:** GitHub won't let you approve your own PR, so requiring 1 would lock you out. You still review every diff in the PR. Later, when you have tests, you'll add "require status checks to pass" here, and that becomes the real gate.
- **Squash-only:** each PR becomes one clean commit on `main`, like `feat: add contact page (#3)`. The detailed work-in-progress commits stay inside the PR.
- **Auto-delete branches:** merged branches are removed, so GitHub doesn't fill up with stale branches.

---

## Step 4: Import the repo into Vercel and deploy

1. Go to vercel.com → **Sign up → Continue with GitHub** and choose the Hobby (free) plan.
2. **Add New → Project → Import Git Repository.** When it asks to install the Vercel GitHub App, choose **"Only select repositories" → `AIDD-course`**. That gives Vercel access to only the repo it needs.
3. Vercel detects **Next.js** automatically. Leave Root Directory as `./`, because the repo root *is* the app. Click **Deploy**.
4. You'll get a production URL like `aidd-course.vercel.app` (Vercel lowercases the name, and adds a suffix if the name is taken). Record it in `CLAUDE.md` and in this plan's Progress log.

**Doing this before Step 3 is finished is fine.** Vercel only needs read access to the repo. The ruleset only controls who can push to `main`, and Vercel never pushes.

**What to check after deploying:**
- The build log in Vercel shows the same steps as your local `npm run build`.
- The production URL loads the default Next.js page.
- On GitHub, the latest commit on `main` gets a ✅ "Vercel" status, and the repo sidebar shows **Deployments → Production**.

**How it works:** the GitHub App makes GitHub notify Vercel on every push. Vercel then clones the commit, runs `npm install` and `next build` on its own Linux servers, and serves the result from its global network.
- Pushes to `main` (the production branch) update the **production** URL.
- Pushes to any other branch create a **preview** deployment with its own URL.

That's the pipeline you prove in Step 5.

---

## Step 4.5: CI quality gate (GitHub Actions + tests)

> **Not started.** This is a learning session: first we go through how GitHub Actions works and what every part of the workflow file does, and only then do we write it. The details below are the outline we agreed on. The exact tools get decided and recorded in the session.

**Goal:** every PR automatically runs these checks, and a PR can't be merged into `main` unless they all pass:

| Check | Command (planned) | Catches |
|---|---|---|
| Lint | `npm run lint` (ESLint) | Code-quality issues and bug patterns, like a missing hook dependency |
| Type-check | `npm run typecheck` → `tsc --noEmit` | Type errors. `next build` type-checks too, but a separate step fails faster and gives clearer output |
| Format check | `npm run format:check` → `prettier --check .` | Code that wasn't formatted, so diffs never contain whitespace-only noise |
| Tests | `npm test` (Vitest + React Testing Library, planned) | Behaviour regressions |
| Build | `npm run build` | Anything that would break the Vercel deploy |

**Topics to learn first (in this order):**
1. **What CI is** and why it's needed: a clean machine runs the checks on every change, so "it works on my PC" is never the only proof.
2. **GitHub Actions concepts:**
   - workflow: a YAML file in `.github/workflows/`
   - events (`on: pull_request`, `on: push`)
   - jobs, and the runners they run on (`ubuntu-latest`)
   - steps, which either use a ready-made action (`uses:`, e.g. `actions/checkout`, `actions/setup-node`) or run a shell command (`run:`)
   - caching `npm` downloads
   - `permissions`, `concurrency` (cancelling outdated runs), and the logs and annotations on the PR
3. **How CI ties into the ruleset:** in `protect-main`, turn on **"Require status checks to pass"** and select the CI job. ⚠️ GitHub only lists a check after it has run at least once, so add it to the ruleset *after* the first CI run.
4. **How CI sits next to Vercel:** both report to the same PR independently. Vercel builds the preview, and CI is the gate.

**What we'll add (one PR, `chore/ci-quality-gate`):**
- **Prettier:** a config file, a `.prettierignore`, `format` and `format:check` scripts, and `eslint-config-prettier` so ESLint and Prettier don't fight over formatting.
- A `typecheck` script.
- **Vitest + React Testing Library + jsdom:** a `vitest.config.mts`, a `test` script and a first real test. See `node_modules/next/dist/docs/01-app/02-guides/testing/vitest.md`. Vitest can't render `async` Server Components, which need E2E tests (Playwright) instead. Playwright is a possible later addition.
- **A pinned Node version** for local, CI and Vercel alike: `.nvmrc` and/or `"engines"` in `package.json`. The two PCs currently run Node 22 and Node 24.
- **`.github/workflows/ci.yml`**, the workflow itself.
- **A testing platform**, to be decided in the session. Candidates:
  - **Codecov:** test coverage reports and a coverage comment on each PR. Free for public repos. This is the leading candidate.
  - **Chromatic:** visual regression testing, but it needs Storybook.
  - **Currents / BrowserStack:** cloud dashboards for Playwright E2E runs.
- Documentation: update `CLAUDE.md` (Commands, Tech stack, Git workflow step 5, Decisions log) and this plan.

**Done when:**
- A PR shows the CI checks running and passing next to the Vercel preview check.
- A deliberately broken PR (a lint error, a type error or a failing test) shows ❌ and can't be merged.
- The testing platform reports on the PR.

---

## Step 5: Prove the pipeline (branch → change → PR → preview → merge → production)

> **With CI in place**, the PR in this step also shows the CI checks. Wait for them to pass (and for the Vercel preview) before merging. Run the same checks locally before pushing: `npm run lint`, `npm run typecheck`, `npm run format:check`, `npm test` and `npm run build`.

```powershell
git switch main
git pull                                      # always branch from the latest main
git switch -c feature/personalize-homepage
```

Make a small visible change:
- In `src/app/page.tsx`, change the main text to your name or the site's purpose.
- In `src/app/layout.tsx`, change `metadata.title`.
- In `CLAUDE.md`, fill in the production URL and check the "Code structure" tree against the real files.

Check it locally before pushing:
```powershell
npm run lint
npm run build      # the same build Vercel runs; if it fails here, it fails there
git add .
git commit -m "feat: personalize homepage and site title"
git push -u origin feature/personalize-homepage
gh pr create --fill                            # or open the PR on github.com
```

Then, on the PR page:
1. The **Vercel bot comments a preview URL** and adds a deployment check. Open the URL and confirm your change is live there, while production is still unchanged.
2. Review the **Files changed** tab like a reviewer would.
3. Click **Squash and merge**, or run `gh pr merge --squash --delete-branch`.
4. Vercel builds `main` and your **production URL updates**. Refresh it to confirm.

Clean up locally:
```powershell
git switch main
git pull
git branch -D feature/personalize-homepage   # skip if gh already deleted it
```
You need `-D` (force delete) because squash merging creates a *new* commit on `main`. Git can't tell your branch's commits were merged, so plain `-d` refuses. Only force delete once the PR shows as merged.

**Bonus proof:** make a commit on `main` and try `git push origin main`. The ruleset should reject it. Then undo the local commit with `git reset --hard origin/main`.

**Clean end state:**
- `git log --oneline` on main shows the scaffold commit and the docs (CLAUDE.md) commit, followed by one squashed commit per PR: docs (progress + CI plan), chore (CI quality gate) and feat (personalize homepage). That's 5 commits with no merge commits. (The original plan expected 3, before the docs and CI PRs were added.)
- `git status` says "nothing to commit, working tree clean".

---

## Step 6: Post your live URL

Share the short production domain (`ai-course-web.vercel.app`). Don't use the long per-deployment URLs with random hashes: those are frozen snapshots of one deploy, while the production domain always shows the latest `main`.

---

## Checklist

- [x] Step 0: git `user.name` fixed, email checked or changed, `init.defaultBranch main`
- [x] Step 1: empty public GitHub repo created and cloned (CLAUDE.md moved aside)
- [x] Step 2: Next.js app scaffolded, runs at localhost:3000
- [x] Step 3a: scaffold + CLAUDE.md commits pushed
- [ ] Step 3b: `protect-main` ruleset + squash-only merge settings (**on the main PC**)
- [ ] Step 4: Vercel project imported and production deploy live
- [ ] Step 4.5: CI workflow (lint, type-check, format, tests, build) + testing platform, and required in the ruleset
- [ ] Step 5: branch → PR → preview → squash merge → production updated
- [ ] Step 6: live URL posted in the group
