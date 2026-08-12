# Chg 的技术笔记

个人技术博客，基于 [Hugo](https://gohugo.io) + [PaperMod](https://github.com/adityatelange/hugo-PaperMod)，自动部署到 GitHub Pages。

线上地址：<https://changhen2004.github.io/chg-blog/>

## 本地预览

```bash
hugo server -D
```

然后打开 <http://localhost:1313/chg-blog/>（新版 Hugo 会按 baseURL 的子路径提供服务）。

## 写新文章

```bash
hugo new posts/文章标题.md
```

编辑 `content/posts/` 下的文件，把开头 `draft: true` 改为 `draft: false`，保存后提交推送即可，GitHub Actions 会自动构建并发布。

## 首次发布步骤

1. 在 GitHub 上创建公开仓库 `chg-blog`（不要勾选 README/.gitignore/license）
2. 在项目目录执行：

```bash
git init
git add -A
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:changhen2004/chg-blog.git
git push -u origin main
```

3. 进入仓库 `Settings → Pages`，把 Source 设为 **GitHub Actions**（构建由工作流完成）
4. 等第一次部署完成（Actions 页面可看进度），访问 <https://changhen2004.github.io/chg-blog/>

## 目录结构

```text
content/posts/      文章（Markdown）
themes/PaperMod/    主题
.github/workflows/  自动部署工作流
hugo.yaml           站点配置
```

## 常见修改

- 博客标题/简介：编辑 `hugo.yaml` 中的 `title`、`homeInfoParams`
- 主题参数（深色模式、目录、分享等）：编辑 `hugo.yaml` 的 `params`
- 想换主题：到 <https://themes.gohugo.io> 选一个，替换 `themes/` 并改 `hugo.yaml`
