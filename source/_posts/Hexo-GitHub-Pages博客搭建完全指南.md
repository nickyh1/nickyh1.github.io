---
title: Hexo + GitHub Pages 博客搭建完全指南
date: 2026-05-01 12:00:00
tags:
  - Hexo
  - GitHub Pages
  - 博客搭建
  - 技术教程
categories:
  - 教程
description: 从零开始搭建 Hexo 博客并部署到 GitHub Pages 的完整教程，涵盖环境准备、主题安装、文章写作、自动部署等全流程。
---

## 一、环境准备

搭建 Hexo 博客需要两个前置工具：**Node.js** 和 **Git**。

**安装 Node.js**：前往 [Node.js 官网](https://nodejs.org/) 下载 LTS（长期支持）版本并安装。安装完成后在终端验证：

```bash
node -v    # 建议 v18+
npm -v
```

**安装 Git**：前往 [Git 官网](https://git-scm.com/) 下载安装。macOS 用户也可以通过 Homebrew 安装（`brew install git`）。安装完成后验证：

```bash
git --version
```

**注册 GitHub 账号**：如果还没有 GitHub 账号，前往 [github.com](https://github.com) 注册一个。

---

## 二、安装 Hexo 并初始化博客

### 2.1 全局安装 Hexo CLI

```bash
npm install -g hexo-cli
```

### 2.2 创建博客项目

```bash
hexo init my-blog     # 初始化项目，my-blog 可以换成你喜欢的名字
cd my-blog
npm install            # 安装依赖
```

初始化完成后，项目目录结构如下：

```
my-blog/
├── _config.yml        # 站点主配置文件（最重要）
├── package.json
├── scaffolds/         # 文章模板
├── source/            # 资源文件夹（文章、图片等）
│   └── _posts/        # 你的博客文章放在这里
└── themes/            # 主题文件夹
```

### 2.3 本地预览

```bash
hexo server
# 或简写
hexo s
```

浏览器打开 `http://localhost:4000`，看到默认的 Hexo 博客页面就说明安装成功了。

---

## 三、基础配置

编辑项目根目录下的 `_config.yml`，修改以下关键配置：

```yaml
# 站点信息
title: My Blog                    # 博客标题
subtitle: 记录学习与生活            # 副标题
description: 一个技术博客           # 站点描述（SEO 用）
keywords: 技术, 编程, 博客          # 关键词
author: YourName                   # 作者名
language: zh-CN                    # 语言设置为中文
timezone: Asia/Shanghai            # 时区

# URL 配置（部署到 GitHub Pages 后需要改）
url: https://你的用户名.github.io
root: /

# 永久链接格式（推荐改成下面这种，更简洁）
permalink: :year/:month/:day/:title/

# 每页显示文章数
index_generator:
  path: ''
  per_page: 10
  order_by: -date
```

---

## 四、安装主题

Hexo 默认主题是 landscape，比较朴素。推荐几个热门主题：

| 主题 | 风格 | 链接 |
|------|------|------|
| Butterfly | 美观、功能丰富 | github.com/jerryc127/hexo-theme-butterfly |
| NexT | 经典简洁 | github.com/next-theme/hexo-theme-next |
| Fluid | 现代、Material Design | github.com/fluid-dev/hexo-theme-fluid |
| Anzhiyu | 二次元风 | github.com/anzhiyu-c/hexo-theme-anzhiyu |

以安装 **Butterfly** 主题为例：

### 4.1 安装主题

```bash
# 在博客根目录下执行
npm install hexo-theme-butterfly
```

### 4.2 启用主题

编辑 `_config.yml`，找到 `theme` 字段并修改：

```yaml
theme: butterfly
```

### 4.3 创建主题配置文件

在博客根目录下创建 `_config.butterfly.yml`，将主题的默认配置复制进去，然后按需修改。这样做的好处是主题升级时不会覆盖你的配置。

### 4.4 安装主题依赖插件

```bash
npm install hexo-renderer-pug hexo-renderer-stylus
```

重新启动本地服务 `hexo s` 即可看到新主题效果。

---

## 五、写文章

### 5.1 创建新文章

```bash
hexo new "我的第一篇博客"
# 会在 source/_posts/ 下生成 "我的第一篇博客.md"
```

### 5.2 文章格式

打开生成的 Markdown 文件，格式如下：

```markdown
---
title: 我的第一篇博客
date: 2026-04-25 14:00:00
tags:
  - 技术
  - Hexo
categories:
  - 教程
cover: /images/cover.jpg       # 文章封面图（可选）
description: 这是一篇关于...    # 文章摘要（可选）
---

正文内容从这里开始，使用标准 Markdown 语法写作。

## 二级标题

这是一段正文。

### 三级标题

- 列表项 1
- 列表项 2
```

### 5.3 其他常用命令

```bash
hexo new page "about"          # 创建「关于」页面
hexo new page "tags"           # 创建标签页
hexo new page "categories"     # 创建分类页
hexo new draft "草稿标题"       # 创建草稿（不会发布）
hexo publish "草稿标题"         # 将草稿发布为正式文章
```

---

## 六、部署到 GitHub Pages

### 方式一：使用 GitHub Actions 自动部署（推荐）

这是目前最推荐的方式，代码推送到 GitHub 后自动构建和部署。

**第一步：创建 GitHub 仓库**

在 GitHub 上新建一个仓库，命名为 `你的用户名.github.io`。如果你想用其他仓库名也可以，但需要额外配置。

**第二步：推送代码到 GitHub**

```bash
cd my-blog
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:你的用户名/你的用户名.github.io.git
git push -u origin main
```

**第三步：创建 GitHub Actions 工作流**

在项目根目录下创建 `.github/workflows/deploy.yml`：

```yaml
name: Deploy Hexo to GitHub Pages

on:
  push:
    branches:
      - main

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Dependencies
        run: npm ci

      - name: Build Hexo
        run: npx hexo generate

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

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

**第四步：启用 GitHub Pages**

进入 GitHub 仓库 → Settings → Pages → Source 选择 **GitHub Actions**。

**第五步：推送并等待部署**

```bash
git add .
git commit -m "add github actions workflow"
git push
```

推送后到仓库的 Actions 标签页查看构建进度，成功后访问 `https://你的用户名.github.io` 即可看到博客。

---

### 方式二：使用 hexo-deployer-git 手动部署

如果不想用 GitHub Actions，也可以用 Hexo 自带的部署插件。

```bash
npm install hexo-deployer-git --save
```

编辑 `_config.yml`，在文件末尾添加：

```yaml
deploy:
  type: git
  repo: git@github.com:你的用户名/你的用户名.github.io.git
  branch: gh-pages
```

然后执行：

```bash
hexo clean && hexo generate && hexo deploy
# 简写
hexo cl && hexo g && hexo d
```

这种方式每次更新都需要手动执行部署命令。

---

## 七、绑定自定义域名（可选）

如果你有自己的域名（比如 `myblog.com`），可以绑定到 GitHub Pages。

**第一步**：在 `source/` 目录下创建一个名为 `CNAME` 的文件（无后缀），内容写你的域名：

```
myblog.com
```

**第二步**：在域名服务商的 DNS 设置中添加记录：

```
类型    主机记录    记录值
CNAME   @          你的用户名.github.io
CNAME   www        你的用户名.github.io
```

**第三步**：在 GitHub 仓库 Settings → Pages → Custom domain 中填入你的域名，并勾选 Enforce HTTPS。

DNS 生效通常需要几分钟到几小时不等。

---

## 八、常用插件推荐

```bash
# 搜索功能
npm install hexo-generator-searchdb

# RSS 订阅
npm install hexo-generator-feed

# Sitemap（SEO 必备）
npm install hexo-generator-sitemap

# 文章字数统计和阅读时长
npm install hexo-wordcount

# 图片懒加载
npm install hexo-lazyload-image
```

安装后在 `_config.yml` 中添加对应配置即可启用。

---

## 九、日常写作工作流

搭建完成后，日常更新博客的流程非常简单：

```bash
# 1. 创建新文章
hexo new "文章标题"

# 2. 用喜欢的编辑器（VS Code、Typora 等）编辑 Markdown 文件

# 3. 本地预览
hexo s

# 4. 满意后推送到 GitHub，自动部署
git add .
git commit -m "新文章：文章标题"
git push
```

---

## 十、常见问题

**Q：hexo s 启动后页面空白或报错？**
先执行 `hexo clean` 清除缓存，再重新 `hexo g && hexo s`。

**Q：部署后页面样式丢失？**
检查 `_config.yml` 中的 `url` 和 `root` 配置是否和实际访问地址一致。

**Q：图片怎么管理？**
可以在 `source/images/` 下存放图片，文章中用 `![](/images/xxx.png)` 引用。也可以使用图床（如 PicGo + GitHub 图床）来管理图片。

**Q：怎么添加评论系统？**
大多数主题都内置了评论插件的配置支持，常用的有 Valine、Waline、Giscus（基于 GitHub Discussions）、Twikoo 等，在主题配置文件中启用即可。

**Q：如何备份博客源码？**
如果使用方式一（GitHub Actions），源码本身就在 GitHub 仓库的 main 分支里，天然就是备份。如果使用方式二（hexo-deployer-git），建议单独建一个分支或仓库存放源码。
