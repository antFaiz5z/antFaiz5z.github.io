# antFaiz5z.github.io

[![Deploy Hexo Site](https://github.com/antFaiz5z/antFaiz5z.github.io/actions/workflows/deploy-pages.yml/badge.svg?branch=whopro)](https://github.com/antFaiz5z/antFaiz5z.github.io/actions/workflows/deploy-pages.yml)
[![Blog](https://img.shields.io/badge/blog-antFaiz5z-blue.svg)](https://antfaiz5z.github.io)
![License](https://img.shields.io/github/license/mashape/apistatus.svg)

这是一个基于 `Hexo` 和 `NexT` 主题构建的个人博客站点。

## 环境

- Node.js: 建议使用 `22`
- npm: 使用仓库自带 `package-lock.json`
- Hexo: `8.1.1`
- Theme: `next-official`

## 安装依赖

```bash
npm install
```

## 本地预览

1. 清理旧的生成结果：

```bash
npm run clean
```

2. 启动本地预览：

```bash
npm run server
```

3. 浏览器打开：

```text
http://localhost:4000
```

如果要监听草稿以外的正常文章内容，这个默认配置已经足够；当前 `render_drafts: false`，不会默认渲染草稿。

## 本地构建

```bash
npm run build
```

生成结果会输出到 `public/` 目录。

## 部署步骤

当前仓库使用 GitHub Actions 自动部署到 GitHub Pages，不依赖本地手工 `hexo deploy`。

### 自动部署

1. 提交修改：

```bash
git add .
git commit -m "更新博客内容"
```

2. 推送到部署分支：

```bash
git push origin whopro
```

3. GitHub Actions 会自动执行以下流程：

- `npm install`
- `npm run build`
- 上传 `public/` 为 Pages artifact
- 部署到 GitHub Pages

工作流文件位于 [deploy-pages.yml](./.github/workflows/deploy-pages.yml)。

### 手动触发部署

也可以在 GitHub Actions 页面手动执行 `Deploy Hexo Site` 工作流。

## 常用命令

```bash
npm run clean
npm run server
npm run build
```

## 当前 Hexo 配置

以下内容来自 [_config.yml](./_config.yml)：

- `theme: next-official`
- `language: zh-CN`
- `url: https://antfaiz5z.github.io/`
- `root: /`
- `permalink: :year/:month/:day/:title/`
- `new_post_name: :title.md`
- `default_layout: post`
- `external_link: true`
- `render_drafts: false`
- `future: true`
- `highlight.enable: true`
- `highlight.line_number: true`
- `per_page: 10`

## 当前 NexT 已开启的设置项

以下内容来自 [_config.next-official.yml](./_config.next-official.yml)：

- `scheme: Gemini`
- `darkmode: true`
- 菜单：`home`、`about`、`archives`、`categories`、`tags`
- `menu_settings.icons: true`
- `menu_settings.badges: true`
- `sidebar.position: left`
- `sidebar.display: always`
- `avatar.rounded: true`
- `site_state: true`
- 社交链接：`GitHub`、`RSS`
- `social_icons.enable: true`
- `toc.enable: true`
- `toc.number: true`
- `reading_progress.enable: true`
- `back2top.enable: true`
- `back2top.scrollpercent: true`
- `bookmark.enable: true`
- `excerpt_description: true`
- `read_more_btn: true`
- `local_search.enable: true`
- 评论系统：`utterances`
- `utterances.enable: true`
- `codeblock.copy_button: true`
- `mermaid.enable: true`

## 目录说明

- `source/`: 文章、页面和静态资源
- `themes/next-official/`: NexT 主题
- `public/`: 本地构建输出
- `.github/workflows/`: GitHub Actions 工作流

## 备注

- 当前部署分支是 `whopro`
- Pages 部署由 GitHub Actions 完成
- Mermaid 图已经通过 NexT 配置启用，可在文章中使用 `{% mermaid ... %}` 标签块
