# 📝 Zhaoyue Zhang's Personal Blog & Resume

A static blog built with Hugo + PaperMod, supporting Markdown writing and auto-deployed to GitHub Pages.

## 🚀 Quick Start

### Prerequisites
- [Hugo](https://gohugo.io/getting-started/installing/) installed
- A GitHub account

### Local Development
```bash
# Clone the repository
git clone https://github.com/zhaoyue-zhang/my-page.git
cd my-page

# Install the theme (if not yet cloned)
git submodule update --init --recursive

# Start the local server
hugo server
```
Open http://localhost:1313 in your browser to preview.

### Create a New Blog Post
```bash
hugo new posts/your-post-title.md
```
Then edit the file at `content/posts/your-post-title.md`.

### Edit the Resume
Edit `content/resume.md`.

## 📁 Project Structure
```
my-page/
├── config.toml            # Site configuration
├── content/
│   ├── posts/             # Blog posts (Markdown)
│   └── resume.md          # Resume page
├── layouts/               # Custom templates (optional overrides)
├── static/
│   ├── css/               # Custom styles
│   └── images/            # Global images
├── themes/
│   └── PaperMod/          # Hugo theme (git submodule)
└── .github/
    └── workflows/
        └── deploy.yml     # GitHub Actions auto-deploy
```

## 🌐 Deploying to GitHub Pages

### First Time Setup
1. Create a GitHub repository: `zhaoyue-zhang/my-page` (public)
2. Push your code to GitHub:
   ```bash
   git add -A
   git commit -m "init blog"
   git push -u origin master
   ```
3. Go to **Settings → Pages** in the repository.
4. Set Source to **Deploy from a branch**, Branch to **master**, Path to **/ (root)**.
5. Click **Save** and wait a minute.
6. Visit `https://zhaoyue-zhang.github.io/my-page/`

### Auto Deploy (Recommended)
`.github/workflows/deploy.yml` is configured with GitHub Actions. Every push to `master` will automatically build and deploy.

## ✨ Features
- 📝 Markdown writing
- 🌙 Dark mode (auto + manual toggle)
- 🔍 Client-side full-text search
- 📚 Tag archive & timeline archive
- 📡 RSS feed
- ⏱ Reading time estimate

## 📌 To-Do
- [ ] Complete resume content
- [ ] Upload profile picture to `static/images/`
- [ ] Add Giscus comment system (optional)
- [ ] Configure a custom domain (optional)