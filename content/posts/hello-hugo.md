---
title: "欢迎来到我的技术博客"
date: 2026-08-12
tags: ["Hugo", "GitHub Pages", "教程"]
showToc: true
---

欢迎！这个博客用 [Hugo](https://gohugo.io) + [PaperMod](https://github.com/adityatelange/hugo-PaperMod) 搭建，托管在 GitHub Pages 上。以后我会在这里记录技术笔记和踩坑经验。

## 如何写一篇新文章

在项目目录下执行：

```bash
hugo new posts/我的第一篇笔记.md
```

然后编辑 `content/posts/` 下的这个文件，把 `draft: true` 改成 `draft: false`，写好内容后保存，接着 `git push` 到 GitHub，剩下的交给 GitHub Actions 自动部署。

## Markdown 示例

### 列表

- 技术笔记
- 踩坑记录
- 小项目

### 代码块

```python
def hello(name: str) -> str:
    return f"Hello, {name}!"

print(hello("Hugo"))
```

### 引用

> 保持好奇，保持记录。

## 小提示

- 写作时可以用 `hugo server -D` 在本地实时预览
- 标签会自动生成 `/tags/` 页面
- 右上角可以切换深色/浅色模式，还有搜索功能
