# Blog Setup

## Initial Setup
```bash
git clone <repo>
npm install -g hexo-cli --force
npm install hexo-deployer-git --save
hexo clean && hexo deploy
```

Visit URL: https://smileusd.github.io/

## Updating the Blog

### Quick Workflow
After adding a new post:
```bash
hexo clean && hexo generate && hexo deploy
```

### Complete Workflow
1. **Create new post:**
   ```bash
   hexo new post "Your Post Title"
   # Edit the generated file in source/_posts/
   ```

2. **Deploy to live site:**
   ```bash
   hexo clean        # Clean previous build
   hexo generate     # Build static site
   hexo deploy       # Deploy to GitHub Pages
   ```

3. **Backup source code:**
   ```bash
   git add .
   git commit -m "Add new post: Your Post Title"
   git push origin master
   ```

### One-Liner Deployment
```bash
hexo clean && hexo generate && hexo deploy && git add . && git commit -m "Add new post" && git push
```

### Available Commands
```bash
hexo new post "Title"        # Create new blog post
hexo new page "about"        # Create new page
hexo clean                   # Clean cache/generated files
hexo generate                # Build static site
hexo server                  # Local preview server (http://localhost:4000)
hexo deploy                  # Deploy to GitHub Pages
```

### File Locations
- **Blog posts**: `source/_posts/*.md`
- **Site config**: `_config.yml`
- **Static files**: `source/` (images, etc.)
- **Live site**: https://smileusd.github.io
