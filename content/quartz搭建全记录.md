# Quartz 搭建个人知识花园全记录

> 用 Quartz + GitHub Pages + 自有域名，搭建一个免费、开源的数字花园，每天记录学习内容。

---

## 前提准备

- Git 环境
- GitHub 账号
- Node.js v22+（我的是 v26.2.0）
- 自己的域名（我用的是 `www.21c.top`）

---

## 第一步：Fork Quartz 仓库

打开 [jackyzha0/quartz](https://github.com/jackyzha0/quartz)，点击右上角 Fork，Fork 到自己的 GitHub 账号下。

我的仓库地址：[github.com/whenStoneHaveLove/quartz](https://github.com/whenStoneHaveLove/quartz)

---

## 第二步：克隆到本地

```bash
git clone git@github.com:whenStoneHaveLove/quartz.git
cd quartz
```

确认 Node.js 版本：

```bash
node -v
```

---

## 第三步：安装依赖

```bash
npm install
```

---

## 第四步：初始化站点

```bash
npx quartz create
```

交互过程中：
- 模板选 **Default (recommended)**，开箱即用
- 后面的确认项一路 Yes 即可

---

## 第五步：本地预览

```bash
npx quartz build --serve
```

浏览器打开 `http://localhost:8080` 可以看到效果。

**我遇到的坑：** `CustomOgImages` 插件 fetch 失败。解决方案：编辑 `quartz.config.yaml`，找到 `og-image` 插件，把 `enabled: true` 改成 `enabled: false`。这个插件只用来生成社交媒体分享预览图，不影响网站核心功能。

---

## 第六步：配置 GitHub Pages 自动部署

Quartz 仓库自带了一份 `.github/workflows/deploy.yml`，但它是面向原项目作者、部署到 Cloudflare 的。我们需要替换成自己的。

编辑 `.github/workflows/deploy.yml`，完整内容如下：

```yaml
name: Deploy Quartz to GitHub Pages

on:
  push:
    branches:
      - v5
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v4
        with:
          node-version: 22
      - name: Install dependencies
        run: npm ci
      - name: Install Quartz plugins
        run: npx quartz plugin install
      - name: Build Quartz
        run: npx quartz build
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: public

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

提交推送：

```bash
git add .github/workflows/deploy.yml
git commit -m "配置 GitHub Pages 自动部署"
git push origin v5
```

---

## 第七步：GitHub Pages 设置

1. 打开仓库的 Settings → Pages
2. Source 选 **GitHub Actions**
3. Custom domain 填 `www.21c.top`，点 Save

---

## 第八步：DNS 域名解析

去域名管理后台（阿里云/腾讯云/Cloudflare/其他），添加 CNAME 记录：

| 记录类型 | 主机记录 | 记录值 |
|---------|---------|--------|
| CNAME | `www` | `whenstonehavelove.github.io` |

注意：记录值末尾不要加斜杠。

---

## 第九步：触发首次部署

打开 [GitHub Actions](https://github.com/whenStoneHaveLove/quartz/actions)，点左边 **Deploy Quartz to GitHub Pages** → **Run workflow** → **Run workflow**。

等待构建完成，DNS 生效后（几分钟到几十分钟），访问 `www.21c.top` 就能看到网站了。

---

## 日常使用流程

### 写笔记

在 `content/` 目录下新建或编辑 `.md` 文件：

```
content/
  ├── index.md              ← 首页
  ├── 每天学一个Linux命令.md
  ├── 前端踩坑记录.md
  └── 读书笔记/
       └── 代码整洁之道.md
```

每篇笔记开头可以加上元信息：

```markdown
---
title: 每天学一个Linux命令
date: 2026-05-29
tags:
  - linux
  - shell
---

今天学了 `grep` 命令...
```

### 本地预览

```bash
npx quartz build --serve
```

打开 `http://localhost:8080` 预览效果。

### 发布上线

```bash
git add .
git commit -m "今日学习笔记"
git push origin v5
```

推送后等待约一分钟，`www.21c.top` 自动更新。

---

## 配合 Obsidian 使用

可以把 Obsidian 库直接指向 `content/` 文件夹，这样 Obsidian 写作和网站发布共享同一套笔记。Quartz 完美支持 Obsidian 的双链 `[[页面名]]`、标签、Callout 等语法。

---

## 配置项速查

| 想改什么 | 改哪里 |
|---------|--------|
| 网站标题 | `quartz.config.yaml` → `pageTitle` |
| 配色 | `quartz.config.yaml` → `colors` |
| 评论系统 | `quartz.config.yaml` → `comments` 插件（配 Giscus） |
| 首页内容 | `content/index.md` |
| 页脚链接 | `quartz.config.yaml` → `footer` 插件 |

---

## 总结

全部零成本（除了域名几十块一年），推送即部署，适合每天记录学习内容、构建个人知识体系。

我的网站：[www.21c.top](https://www.21c.top)
