# 📝 照悦的博客 & 简历 / Zhaoyue's Blog & Résumé

A static blog built with Hugo + PaperMod, deployed to GitHub Pages via GitHub Actions.
Hugo + PaperMod 构建的静态博客与简历站点，通过 GitHub Actions 自动部署到用户站根路径。

🌐 **Live site**: <https://zhaoyue-zhang.github.io/>

## ✨ Features / 特性

- 📝 Markdown 写作，语法高亮与表格
- 🌙 深色模式（跟随系统 + 手动切换）
- 🌍 中英文双语切换（i18n，前端记忆偏好）
- 📊 阅读量统计（不蒜子 busuanzi，本文阅读量 + 站点总访问量）
- ⏱ 阅读时长估算
- 📄 内置简历页（`content/resume.md`）
- 🚀 推送 `main` 分支即自动构建部署

## 🚀 Quick Start / 快速开始

### Prerequisites / 前置条件
- [Hugo](https://gohugo.io/getting-started/installing/) (extended 版本)

### Local Development / 本地开发
```bash
# 克隆仓库（用户站仓即源码仓）
git clone https://github.com/zhaoyue-zhang/zhaoyue-zhang.github.io.git
cd zhaoyue-zhang.github.io

# 初始化主题子模块
git submodule update --init --recursive

# 启动本地预览
hugo server
```
Open http://localhost:1313 in your browser to preview.

### Create a New Post / 新建文章
```bash
hugo new posts/your-post-title.md
```
然后编辑 `content/posts/your-post-title.md`。也可从 Notion 等导出 Markdown 后填入 front matter 放进 `content/posts/`。

### Edit the Résumé / 编辑简历
编辑 `content/resume.md`（章节用 `###`，列表用 Markdown 标准语法）。

## 📁 Project Structure / 项目结构
```
zhaoyue-zhang.github.io/
├── config.toml            # 站点配置（baseurl / 菜单 / 社交链接）
├── content/
│   ├── posts/             # 博客文章 (Markdown)
│   └── resume.md          # 简历页
├── layouts/               # 自定义模板（覆盖主题）
│   ├── _default/          # baseof / single / list
│   ├── resume/            # 简历页模板
│   └── index.html         # 首页
├── static/
│   ├── css/               # 自定义样式
│   └── images/            # 全局图片
├── themes/
│   └── PaperMod/          # Hugo 主题 (git submodule)
└── .github/
    └── workflows/
        └── deploy.yml     # GitHub Actions 自动部署
```

## 🌐 Deployment / 部署

本项目使用 GitHub Actions 自动部署到 GitHub 用户站 `zhaoyue-zhang.github.io`。

- **仓库**：`zhaoyue-zhang/zhaoyue-zhang.github.io`（源码与 Pages 同仓）
- **分支**：`main`（`.github/workflows/deploy.yml` 监听 `main` 分支的 push）
- **流程**：推送到 `main` → Actions 用 Hugo 构建产物 → 发布到 GitHub Pages
- **访问**：<https://zhaoyue-zhang.github.io/>（用户站根路径）

```bash
git add -A
git commit -m "your message"
git push origin main
```

## 📌 To-Do / 待办
- [x] 完善简历内容（个人信息 / 教育背景 / 工作经历 / 技能）
- [ ] 补充简历的项目经历与自我评价
- [ ] 上传头像到 `static/images/`
- [ ] 接入评论系统 Giscus（可选）
- [ ] 配置自定义域名（可选）
