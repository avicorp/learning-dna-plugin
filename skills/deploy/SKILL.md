---
name: deploy
description: Deploy your learning app to a free hosting service (GitHub Pages, Netlify, or Vercel)
disable-model-invocation: true
argument-hint: "[github-pages|netlify|vercel]"
---

# /deploy — Deploy Your Learning App

## Usage
`/learning-dna:deploy` — choose a platform interactively
`/learning-dna:deploy github-pages` — deploy to GitHub Pages
`/learning-dna:deploy netlify` — deploy to Netlify
`/learning-dna:deploy vercel` — deploy to Vercel

## Flow

> **Self-driving pipeline:** After each step completes, proceed immediately to the next. Do NOT pause between steps unless the step explicitly requires user input. Mandatory user interaction points: platform selection (Step 2), platform-specific configuration questions (Step 3). All other transitions are automatic.

### Step 1: Precondition Gate

1. Check that `learning-app/` directory exists. If missing → **STOP.** Tell the user:
   ```
   No learning app found. Create one first:
   /learning-dna:new-topic <topic-name>
   ```
2. Run the build:
   ```bash
   cd learning-app && npm run build 2>&1
   ```
   - If build fails → **STOP.** Show the error and tell the user:
     ```
     The app has build errors. Fix them first, or rebuild:
     /learning-dna:build-app
     ```
3. Verify `learning-app/dist/index.html` exists. If missing → STOP with the same message as above.

### Step 2: Platform Selection (MANDATORY unless argument provided)

If the user passed a valid argument (`github-pages`, `netlify`, or `vercel`), skip this step and use that selection.

Otherwise, use **AskUserQuestion** to present the three options:

| # | header | question | options |
|---|--------|----------|---------|
| 1 | Deploy Platform | Where would you like to deploy your learning app? All three options are completely free. | **GitHub Pages** — "Free hosting tied to your GitHub repo. Best if your code is already on GitHub. URL: `https://<username>.github.io/<repo>/`" / **Netlify** — "Free tier with 100 GB bandwidth/month. Easiest setup — drag-and-drop or CLI. URL: `https://<site-name>.netlify.app`" / **Vercel** — "Free for personal projects. Fast global CDN. URL: `https://<project>.vercel.app`" |

Proceed to the matching Step 3 branch.

---

### Step 3A: GitHub Pages

#### 3A.1 — Repository Name

Use **AskUserQuestion**:

| header | question |
|--------|----------|
| Repository Name | What is your GitHub repository name? This sets the URL base path — e.g., if your repo is `my-learning`, the app will live at `https://<username>.github.io/my-learning/`. |

Capture the answer as `{repo-name}`.

#### 3A.2 — Vite Base Path

1. Read `learning-app/vite.config.ts`
2. Set the `base` property to `'/{repo-name}/'`
   - If a `base` property already exists, update it
   - If no `base` property exists, add it inside the `defineConfig({})` call
3. Save the file

#### 3A.3 — Routing Strategy

The generated app uses React Router with `BrowserRouter`. GitHub Pages is a static file host that does not support server-side routing — all paths except `/` return 404.

Use **AskUserQuestion**:

| # | header | question | options |
|---|--------|----------|---------|
| 1 | Routing | GitHub Pages doesn't support client-side routing natively. How should we handle this? | **Hash Router** — "Switch to HashRouter. URLs will include `#` (e.g., `/#/kubernetes`). Simplest and most reliable option (recommended)." / **404 Redirect** — "Add a `404.html` redirect workaround. Keeps clean URLs but may have edge cases with some browsers." |

**If Hash Router:**
1. Find the file that imports `BrowserRouter` from `react-router-dom` (typically `src/main.tsx` or `src/App.tsx`)
2. Replace `BrowserRouter` with `HashRouter` in both the import and the JSX
3. No other changes needed

**If 404 Redirect:**
1. Create `learning-app/public/404.html`:
   ```html
   <!DOCTYPE html>
   <html>
     <head>
       <meta charset="utf-8">
       <title>Redirecting…</title>
       <script>
         // Single-page app redirect for GitHub Pages
         // Stores the path in sessionStorage and redirects to index.html
         sessionStorage.setItem('redirect', location.pathname + location.search + location.hash);
         location.replace(location.origin + '/{repo-name}/');
       </script>
     </head>
     <body></body>
   </html>
   ```
2. Add a redirect restore script to `learning-app/index.html`, inside `<body>` before the root div:
   ```html
   <script>
     (function() {
       var redirect = sessionStorage.getItem('redirect');
       if (redirect) {
         sessionStorage.removeItem('redirect');
         history.replaceState(null, '', redirect);
       }
     })();
   </script>
   ```

#### 3A.4 — Deployment Method

Use **AskUserQuestion**:

| # | header | question | options |
|---|--------|----------|---------|
| 1 | Deploy Method | How would you like to deploy to GitHub Pages? | **GitHub Actions** — "Automated deploy on every push to main. I'll create the workflow file for you (recommended)." / **Manual** — "Deploy manually using the `gh-pages` npm package. Run a command each time you update." |

**If GitHub Actions:**

Create `.github/workflows/deploy-learning-app.yml` in the **project root** (not inside `learning-app/`):

```yaml
name: Deploy Learning App to GitHub Pages

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
  build-and-deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: learning-app/package-lock.json

      - name: Install dependencies
        run: cd learning-app && npm ci

      - name: Build
        run: cd learning-app && npm run build

      - uses: actions/configure-pages@v5

      - uses: actions/upload-pages-artifact@v3
        with:
          path: learning-app/dist

      - id: deployment
        uses: actions/deploy-pages@v4
```

Then tell the user:
```
✓ GitHub Actions workflow created: .github/workflows/deploy-learning-app.yml

Next steps:
1. Push your code to GitHub:
   git add -A && git commit -m "Add GitHub Pages deployment" && git push

2. Enable GitHub Pages in your repo settings:
   → https://github.com/{owner}/{repo-name}/settings/pages
   → Source: select "GitHub Actions"

3. Your app will deploy automatically on every push to main
   → URL: https://{owner}.github.io/{repo-name}/
```

**If Manual:**

1. Install the `gh-pages` package:
   ```bash
   cd learning-app && npm install --save-dev gh-pages
   ```
2. Read `learning-app/package.json` and add a `"deploy"` script:
   ```json
   "scripts": {
     "deploy": "gh-pages -d dist"
   }
   ```
3. Rebuild:
   ```bash
   cd learning-app && npm run build
   ```
4. Tell the user:
   ```
   ✓ Manual deploy configured!

   Next steps:
   1. Make sure your code is pushed to GitHub

   2. Enable GitHub Pages in your repo settings:
      → https://github.com/{owner}/{repo-name}/settings/pages
      → Source: select "Deploy from a branch"
      → Branch: select "gh-pages" / root

   3. Deploy now:
      cd learning-app && npm run deploy

   4. Your app will be live at:
      → https://{owner}.github.io/{repo-name}/

   To update after changes: rebuild and redeploy:
      cd learning-app && npm run build && npm run deploy
   ```

→ Proceed to Step 4.

---

### Step 3B: Netlify

#### 3B.1 — Vite Base Path Reset

1. Read `learning-app/vite.config.ts`
2. If `base` is set to anything other than `'/'`, reset it to `'/'` (or remove the `base` property entirely)
3. If changed, rebuild: `cd learning-app && npm run build`

#### 3B.2 — Create Netlify Configuration

Create `learning-app/netlify.toml`:
```toml
[build]
  publish = "dist"
  command = "npm run build"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

The `[[redirects]]` block handles SPA client-side routing — all paths serve `index.html` so React Router works correctly.

#### 3B.3 — Deployment Method

Use **AskUserQuestion**:

| # | header | question | options |
|---|--------|----------|---------|
| 1 | Deploy Method | How would you like to deploy to Netlify? | **Drag and Drop** — "Upload your `dist/` folder directly in the browser. Fastest way to get started — no account setup needed beyond sign-up (recommended for first-time users)." / **Netlify CLI** — "Deploy from your terminal. Great for repeated deploys." / **Git Integration** — "Connect your GitHub repo for automatic deploys on every push." |

**If Drag and Drop:**
```
✓ Netlify config created: learning-app/netlify.toml

Next steps:
1. Open Netlify Drop in your browser:
   → https://app.netlify.com/drop

2. Drag the entire folder: learning-app/dist/
   (Tip: open your file manager, navigate to learning-app/dist/, and drag it onto the page)

3. Netlify will give you a live URL instantly (e.g., https://random-name.netlify.app)

4. Optional: claim the site to customize the URL and enable continuous deploys
   → Sign up / log in at https://app.netlify.com
```

**If Netlify CLI:**
```
✓ Netlify config created: learning-app/netlify.toml

Next steps:
1. Install the Netlify CLI (if not already installed):
   npm install -g netlify-cli

2. Log in to Netlify:
   netlify login

3. Deploy your app:
   cd learning-app && netlify deploy --prod --dir=dist

4. The CLI will prompt you to link or create a site — follow the prompts

5. Your app URL will be shown after deploy completes

Docs: https://docs.netlify.com/cli/get-started/
```

**If Git Integration:**
```
✓ Netlify config created: learning-app/netlify.toml

Next steps:
1. Push your code to GitHub (if not already):
   git add -A && git commit -m "Add Netlify deployment config" && git push

2. Go to Netlify and create a new site:
   → https://app.netlify.com/start

3. Choose "Import an existing project" → select your GitHub repo

4. Configure build settings:
   - Base directory: learning-app
   - Build command: npm run build
   - Publish directory: learning-app/dist

5. Click "Deploy site" — Netlify will build and deploy automatically

6. Every future push to your main branch triggers a new deploy

Docs: https://docs.netlify.com/site-deploys/create-deploys/
```

→ Proceed to Step 4.

---

### Step 3C: Vercel

#### 3C.1 — Vite Base Path Reset

1. Read `learning-app/vite.config.ts`
2. If `base` is set to anything other than `'/'`, reset it to `'/'` (or remove the `base` property entirely)
3. If changed, rebuild: `cd learning-app && npm run build`

#### 3C.2 — Create Vercel Configuration

Create `learning-app/vercel.json`:
```json
{
  "rewrites": [
    { "source": "/(.*)", "destination": "/index.html" }
  ]
}
```

The `rewrites` rule handles SPA client-side routing — all paths serve `index.html` so React Router works correctly.

#### 3C.3 — Deployment Method

Use **AskUserQuestion**:

| # | header | question | options |
|---|--------|----------|---------|
| 1 | Deploy Method | How would you like to deploy to Vercel? | **Vercel CLI** — "Deploy from your terminal with a single command. Fast and simple (recommended)." / **Git Integration** — "Connect your GitHub repo for automatic deploys on every push." |

**If Vercel CLI:**
```
✓ Vercel config created: learning-app/vercel.json

Next steps:
1. Install the Vercel CLI (if not already installed):
   npm install -g vercel

2. Deploy your app:
   cd learning-app && vercel --prod

3. First-time users will be prompted to:
   - Log in or create an account
   - Link the project to a Vercel project
   - Confirm the settings (defaults are fine for Vite apps)

4. Your app URL will be shown after deploy completes
   (e.g., https://your-project.vercel.app)

Docs: https://vercel.com/docs/cli
```

**If Git Integration:**
```
✓ Vercel config created: learning-app/vercel.json

Next steps:
1. Push your code to GitHub (if not already):
   git add -A && git commit -m "Add Vercel deployment config" && git push

2. Go to Vercel and import your project:
   → https://vercel.com/new

3. Select your GitHub repository

4. Configure the project:
   - Framework Preset: Vite
   - Root Directory: learning-app
   (Vercel auto-detects most settings from vite.config.ts)

5. Click "Deploy" — Vercel will build and deploy automatically

6. Every future push to your main branch triggers a new deploy

Docs: https://vercel.com/docs/deployments/git
```

→ Proceed to Step 4.

---

### Step 4: Deployment Verification (LOOP — max 3 attempts)

1. Use **AskUserQuestion** to ask the user for their deployed URL:

   | header | question |
   |--------|----------|
   | Verify Deployment | Paste your deployed app URL so I can verify it's working (e.g., `https://yoursite.netlify.app`). Or type "skip" if you haven't deployed yet. |

2. If the user types "skip" or similar → skip verification and go to Step 5 with a note that verification was skipped.

3. If a URL is provided, use **WebFetch** to check the deployed app:
   - Fetch the URL
   - Verify the response contains HTML with the expected app markers (root div, script tags, "Learning DNA" or topic name)
   - If verification succeeds → proceed to Step 5
   - If verification fails → diagnose the likely cause:

   | Symptom | Likely Cause | Fix |
   |---------|-------------|-----|
   | 404 Not Found | Site not deployed yet or wrong URL | Wait a few minutes for deploy to complete, double-check URL |
   | Blank white page | Vite `base` path misconfigured | Check `vite.config.ts` base matches the deployment URL path |
   | Page loads but routes return 404 | SPA routing not configured | Verify `netlify.toml` redirects, `vercel.json` rewrites, or HashRouter for GitHub Pages |
   | Assets (JS/CSS) fail to load | `base` path wrong | Inspect the HTML `<script>` and `<link>` `src`/`href` paths — they must match the deployment root |

   - Report the issue and suggested fix, then ask for the URL again (loop)
   - After 3 failed attempts, provide the diagnostic table above and tell the user to check manually

### Step 5: Success Report

Report the deployment result:

```
✓ Your learning app is ready to share!

  Platform: {platform}
  URL: {deployed-url or "deploy pending — follow the steps above"}

  The app is fully static — no backend or database needed.
  All user progress is saved locally in each visitor's browser (IndexedDB).

  To update after adding new topics:
  1. /learning-dna:new-topic <topic>   → add content
  2. /learning-dna:build-app           → rebuild the app
  3. /learning-dna:deploy {platform}   → redeploy
```

## Idempotent Re-runs

When `/learning-dna:deploy` is run again on a project that was previously deployed:

1. **Detect existing config** — check for `.github/workflows/deploy-learning-app.yml`, `learning-app/netlify.toml`, or `learning-app/vercel.json`
2. If config exists for the same platform the user selects → skip configuration steps, just rebuild and provide redeploy instructions
3. If config exists for a **different** platform → warn the user that switching platforms will replace the existing config, ask for confirmation before proceeding
4. Always rebuild `learning-app/dist/` before redeploying to ensure latest content is included

## Key Constraints

- **Never skip the precondition gate** — the app must exist and build cleanly before attempting any deployment
- **Never modify the app's source code beyond what's needed for deployment** — only change `vite.config.ts` base path, router type (if user chooses HashRouter), and add deployment config files
- **Always provide specific URLs and links** — every instruction must include the exact URL the user needs to visit
- **Always explain why** — when creating config files or changing settings, briefly explain what each change does
- **Respect user choice** — never auto-select a platform or deployment method
