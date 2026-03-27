# YeeSkyTrail 博客使用手册

本手册基于 **Hugo Theme Stack** 主题，旨在帮助你快速开始在 YeeSkyTrail 博客上撰写和发布内容。

---

## 1. 项目简介
YeeSkyTrail 是一个基于 **Hugo** 静态网站生成器构建的个人博客。
- **主题**: [Hugo Theme Stack](https://github.com/CaiJimmy/hugo-theme-stack)
- **主要配置**: 位于 `config/_default/` 目录下。
- **自动化**: 配置了 GitHub Actions，推送到 GitHub 后会自动部署。

---

## 2. 撰写新文章
项目推荐使用 **Page Bundle** 模式管理文章，即每篇文章拥有独立文件夹。

### 第一步：创建文章文件夹
在 `content/post/` 目录下创建一个文件夹，建议使用英文或拼音命名（作为 URL 的一部分）：
```bash
mkdir -p content/post/my-new-post
```

### 第二步：创建内容文件
在新建的文件夹内创建 `index.md`：
```bash
touch content/post/my-new-post/index.md
```

### 第三步：填写元数据 (Front Matter)
在 `index.md` 顶部添加以下配置信息：
```markdown
---
title: "文章标题"
description: "简短的文章描述"
date: 2026-03-27T12:00:00+08:00
image: cover.jpg           # 可选：文章封面图
categories:                # 分类
    - 生活
tags:                      # 标签
    - 随笔
slug: my-new-post          # 自定义 URL 后缀
---

这里开始写你的 Markdown 内容...
```

### 第四步：添加图片（可选）
如果你在元数据中设置了 `image: cover.jpg`，请将图片文件直接存放在 `content/post/my-new-post/` 文件夹下。

---

## 3. 本地预览
如果你本地安装了 Hugo，可以使用以下命令启动实时预览服务器：

```bash
hugo server
```
- 访问地址: `http://localhost:1313`
- 预览模式下，修改保存文件后浏览器会自动刷新。

---

## 4. 发布文章
当你完成写作并准备好发布时，执行以下 Git 操作：

```bash
git add .
git commit -m "feat: 新增文章 - 文章标题"
git push
```
GitHub Actions 会自动触发构建任务，大约 1-2 分钟后，你的新文章就会出现在线上博客中。

---

## 5. 进阶参考
- **示例文章**: 查看 `backup/content/post/` 目录下的示例，了解如何使用图片画廊、数学公式和短代码。
- **修改配置**: 网站标题、菜单、作者信息等可以在 `config/_default/` 下的配置文件中修改。
- **主题文档**: 访问 [Stack 主题官方文档](https://stack.jimmycai.com/) 获取更多高级功能。
