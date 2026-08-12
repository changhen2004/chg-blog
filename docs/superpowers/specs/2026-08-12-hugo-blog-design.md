# Hugo 个人技术博客设计文档

- 日期：2026-08-12
- 状态：已确认

## 目标

搭建一个用于记录技术笔记的个人博客，发布到 GitHub Pages。

## 技术选型

- 静态站点生成器：Hugo（extended）
- 主题：PaperMod（自带搜索、深色模式、标签/归档、代码复制）
- 托管：GitHub Pages，仓库 `changhen2004/chg-blog`，地址 `https://changhen2004.github.io/chg-blog/`
- 部署：GitHub Actions 自动构建（push 到 main 触发）

## 站点结构

- `content/posts/`：文章，Markdown 格式，front matter 含 title/date/tags
- `content/search.md`、`content/archives.md`：搜索页与归档页（PaperMod 内置 layout）
- `hugo.yaml`：站点配置（baseURL、标题、菜单、主题参数）
- `.github/workflows/deploy.yml`：构建 + 部署工作流

## 写作流程

1. `hugo new posts/<标题>.md` 创建草稿
2. 本地 `hugo server -D` 预览，`draft` 改为 `false`
3. `git push`，Actions 自动构建发布

## 明确不做（YAGNI）

- 评论系统：暂不引入，后续可按需加 giscus
- 自定义主题/复杂页面：用 PaperMod 默认能力
- 自定义域名：暂用 GitHub 默认域名
