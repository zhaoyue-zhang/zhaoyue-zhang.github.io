# 📝 赵越的个人博客 & 简历

基于 Hugo + PaperMod 主题搭建的静态博客，支持 Markdown 写作，自动部署到 GitHub Pages。

## 🚀 快速开始

### 前置条件
- 安装 Hugo：https://gohugo.io/getting-started/installing/
- GitHub 账号

### 本地开发
```bash
# 克隆仓库
git clone https://github.com/zhaoyue-zhang/my-page.git
cd my-page

# 安装主题（如果尚未克隆）
git submodule update --init --recursive

# 启动本地服务器
hugo server
```
浏览器打开 http://localhost:1313 即可预览。

### 新建博客文章
```bash
hugo new posts/你的文章标题.md
```
然后编辑 `content/posts/你的文章标题.md`。

### 修改简历
编辑 `content/resume.md`。

## 📁 项目结构
```
my-page/
├── config.toml            # 站点配置
├── content/
│   ├── posts/             # 博客文章（Markdown）
│   └── resume.md          # 简历页面
├── layouts/               # 自定义模板（可选覆盖主题）
├── static/
│   └── css/               # 自定义样式
│   └── images/            # 全局图片
├── themes/
│   └── PaperMod/          # Hugo 主题（git submodule）
└── .github/
    └── workflows/
        └── deploy.yml     # GitHub Actions 自动部署
```

## 🌐 部署到 GitHub Pages

### 第一次部署
1. 在 GitHub 上创建仓库 `zhaoyue-zhang/zhaoyue-zhang.github.io`
2. 推送代码到 GitHub：
   ```bash
   git add -A
   git commit -m "init blog"
   git push -u origin master
   ```
3. 进入仓库 **Settings → Pages**，选择部署分支为 `master`，路径为 `/ (root)`。
4. 访问 `https://zhaoyue-zhang.github.io/my-page/`

### 自动部署（推荐）
`.github/workflows/deploy.yml` 配置了 GitHub Actions，每次 push 到 `master` 分支会自动构建并部署。

## ✨ 功能
- 📝 Markdown 写作
- 🌙 暗黑模式
- 🔍 客户端全文搜索
- 📚 标签归档 & 时间归档
- 📡 RSS 订阅
- ⏱ 阅读时间估算

## 📌 待办
- [ ] 完善简历内容
- [ ] 上传个人头像到 `static/images/`
- [ ] 添加 Giscus 评论系统（可选）
- [ ] 配置自定义域名（可选）
