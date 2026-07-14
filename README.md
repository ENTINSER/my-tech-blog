# Mingrun Tech Blog

基于 Hugo + PaperMod 的个人技术博客，使用 GitHub Pages 免费托管。

## 本地预览

```bash
cd /Users/mingrun/mingrun-tech-blog
hugo server -D
```

访问 http://localhost:1313 预览。

## 新增文章

```bash
hugo new content posts/my-new-post/index.md
```

编辑 `content/posts/my-new-post/index.md` 即可。

## 发布到 GitHub Pages

1. 在 GitHub 创建空仓库 `mingrun-tech-blog`。
2. 在本地目录初始化并推送：

```bash
cd /Users/mingrun/mingrun-tech-blog
git init
git add .
git commit -m "init: hugo + papermod + two posts"
git branch -M main
git remote add origin https://github.com/mingrun/mingrun-tech-blog.git
git push -u origin main
```

3. 进入 GitHub 仓库 Settings → Pages → Source，选择 **GitHub Actions**。
4. 等待 Actions 运行完成，访问 `https://mingrun.github.io/mingrun-tech-blog/`。

## 绑定独立域名（可选）

1. 在 DNS 添加 CNAME 记录：`blog.yourdomain.com` → `mingrun.github.io`
2. 在仓库根目录创建 `static/CNAME`，内容为 `blog.yourdomain.com`
3. 在 `hugo.toml` 中把 `baseURL` 改为 `https://blog.yourdomain.com/`
4. 推送后等待 DNS 生效

## 目录结构

```text
.
├── .github/workflows/hugo.yml   # GitHub Actions 自动部署
├── archetypes/                  # 文章模板
├── assets/                      # 需要 Hugo 处理的资源
├── content/                     # 博客文章
├── layouts/                     # 自定义布局
├── static/                      # 静态资源
├── themes/PaperMod              # 主题
├── hugo.toml                    # 站点配置
└── README.md
```

## 配置说明

- 站点标题、菜单、社交链接：修改 `hugo.toml`
- 主题参数：参考 [PaperMod Wiki](https://github.com/adityatelange/hugo-PaperMod/wiki)
- 评论：推荐使用 [Giscus](https://giscus.app/)（基于 GitHub Discussions）
