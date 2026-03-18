# GitHub Pages Deployment Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Configure Hexo blog to deploy to GitHub Pages at https://smileusd.github.io

**Architecture:** Two-repository setup with source code in `smileusd/blog` and built static files deployed to `smileusd/smileusd.github.io` using hexo-deployer-git

**Tech Stack:** Hexo static site generator, hexo-deployer-git, GitHub Pages

---

## Task 1: Update Site Configuration

**Files:**
- Modify: `_config.yml:16` (URL configuration)
- Modify: `_config.yml:105` (deployment repository)

**Step 1: Update site URL**

In `_config.yml`, change line 16 from:
```yaml
url: http://example.com
```
to:
```yaml
url: https://smileusd.github.io
```

**Step 2: Update deployment repository to HTTPS**

In `_config.yml`, change line 105 from:
```yaml
repo: git@github.com:smileusd/smileusd.github.io.git
```
to:
```yaml
repo: https://github.com/smileusd/smileusd.github.io.git
```

**Step 3: Commit configuration changes**

```bash
git add _config.yml
git commit -m "config: update site URL and deployment repo for GitHub Pages"
```

---

## Task 2: Verify Build Process

**Files:**
- Monitor: `public/` directory
- Monitor: `db.json` (Hexo database file)

**Step 1: Clean previous build**

Run: `npm run clean`
Expected: "INFO  Deleted database." and `public/` directory removed

**Step 2: Generate static site**

Run: `npm run build`
Expected:
- "INFO  Start processing"
- "INFO  Files loaded in [time]"
- "INFO  Generated: [number] files"
- `public/` directory created with HTML/CSS/JS files

**Step 3: Verify generated files**

Run: `ls -la public/`
Expected: See files like `index.html`, `css/`, `js/`, and blog post directories

**Step 4: Check site configuration in generated files**

Run: `grep "smileusd.github.io" public/index.html`
Expected: Find references to the new URL in the generated HTML

---

## Task 3: Test Local Server (Optional Verification)

**Files:**
- Monitor: Local server at `http://localhost:4000`

**Step 1: Start development server**

Run: `npm run server`
Expected: "INFO  Hexo is running at http://localhost:4000"

**Step 2: Verify site loads locally**

Open browser to `http://localhost:4000`
Expected: Site loads with correct styling and content

**Step 3: Stop development server**

Press: `Ctrl+C`
Expected: Server stops

---

## Task 4: Deploy to GitHub Pages

**Files:**
- Target: `https://github.com/smileusd/smileusd.github.io.git`
- Monitor: `.deploy_git/` directory

**Step 1: Deploy to GitHub Pages**

Run: `npm run deploy`
Expected:
- "INFO  Deploying: git"
- "INFO  Clearing .deploy_git folder..."
- "INFO  Copying files from public folder..."
- Git operations showing push to remote repository

**Step 2: Verify deployment repository was updated**

Check: `ls -la .deploy_git/`
Expected: See git repository with built files

**Step 3: Verify remote repository received update**

Run: `cd .deploy_git && git log -1 --oneline && cd ..`
Expected: See recent commit with timestamp

---

## Task 5: Verify Live Site

**Files:**
- Monitor: https://smileusd.github.io (live site)

**Step 1: Wait for GitHub Pages deployment**

Wait: 2-3 minutes for GitHub Pages to process the update
Note: GitHub Pages needs time to build and serve the updated content

**Step 2: Check site accessibility**

Run: `curl -I https://smileusd.github.io`
Expected: HTTP 200 response with GitHub Pages headers

**Step 3: Verify content appears correctly**

Visit: https://smileusd.github.io in browser
Expected:
- Site loads with correct styling
- Blog posts visible (including Linux networking series)
- Navigation works properly
- No broken links or missing assets

---

## Task 6: Commit and Push Source Changes

**Files:**
- Modify: Git repository state

**Step 1: Push source code changes**

Run: `git push origin master`
Expected: Successfully pushes updated _config.yml to source repository

**Step 2: Verify both repositories are in sync**

Check source repo: `git status`
Expected: "Your branch is up to date with 'origin/master'"

**Step 3: Document successful deployment**

Note: Site is now live at https://smileusd.github.io with:
- Updated configuration
- Working deployment pipeline
- All content properly served

---

## Verification Checklist

After completion, verify:
- [ ] Site loads at https://smileusd.github.io
- [ ] All blog posts display correctly
- [ ] CSS and JavaScript assets load properly
- [ ] Navigation links work
- [ ] Future deployments work with `npm run deploy`
- [ ] Source code is backed up in `smileusd/blog` repository