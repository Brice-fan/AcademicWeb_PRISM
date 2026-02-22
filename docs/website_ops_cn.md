# AcademicWeb_PRISM 网站部署与运维手册（GitHub Pages + Cloudflare + 自定义域名）

## 1. 目标与架构

本文档用于记录 `AcademicWeb_PRISM` 网站的可复用搭建与维护流程，适用于：

- 新项目部署到 GitHub Pages
- 域名在腾讯云购买，但 DNS 托管在 Cloudflare
- 日常内容修改后推送发布
- 常见故障排查（Workflow 不触发、域名不生效、资源 404 等）

当前推荐架构：

- 代码托管：GitHub (`main` 分支)
- 构建发布：GitHub Actions -> GitHub Pages
- 域名注册商：腾讯云
- DNS 托管：Cloudflare

---

## 2. 本项目关键配置（必须存在）

### 2.1 Next.js 静态导出配置

文件：`next.config.ts`

需要包含：

```ts
const nextConfig: NextConfig = {
  output: 'export',
  trailingSlash: true,
  images: {
    unoptimized: true,
  },
};
```

说明：

- `output: 'export'`：生成纯静态站点（`out/`），供 GitHub Pages 托管
- `images.unoptimized: true`：禁用 Next Image 优化服务，避免静态站点图片失效

### 2.2 GitHub Actions 部署工作流

文件：`.github/workflows/deploy.yml`

关键触发：

```yml
on:
  push:
    branches:
      - main
  workflow_dispatch:
```

说明：

- `push`：每次推送 `main` 自动部署
- `workflow_dispatch`：支持在 Actions 页面手动触发

### 2.3 自定义域名文件

文件：`public/CNAME`

内容：

```txt
bingfan.work
```

说明：

- 构建后会进入 `out/CNAME`
- GitHub Pages 每次部署都会识别该域名

---

## 3. GitHub Pages 初始设置流程

1. 进入仓库 `Settings -> Pages`
2. `Source` 选择 `GitHub Actions`
3. `Custom domain` 填入：`bingfan.work`
4. 暂时不要着急勾 `Enforce HTTPS`，等 DNS 与证书就绪后再开

---

## 4. Cloudflare DNS 配置（最关键）

> 前提：域名 `bingfan.work` 的 NS 已托管到 Cloudflare。

在 Cloudflare 的 DNS 里设置如下记录：

1. `@` -> `A` -> `185.199.108.153`
2. `@` -> `A` -> `185.199.109.153`
3. `@` -> `A` -> `185.199.110.153`
4. `@` -> `A` -> `185.199.111.153`
5. `www` -> `CNAME` -> `brice-fan.github.io`

操作提示：

- 先删除旧的 Netlify 记录（例如指向 `*.netlify.app` 的 `@` / `www`）
- 删除错误记录（例如 `8.8.8.8` 这种并非站点服务器 IP）
- 对 GitHub Pages 记录，建议先设为 **DNS only（灰云）**，稳定后再评估是否代理

校验命令：

```bash
dig +short bingfan.work A
dig +short www.bingfan.work CNAME
```

预期：

- 根域返回 4 个 `185.199.10x.153`
- `www` 返回 `brice-fan.github.io.`

---

## 5. 第一次发布检查清单

1. `deploy.yml` 已在远端默认分支（通常是 `main`）
2. `public/CNAME` 已在远端
3. `Settings -> Actions -> General` 权限允许执行 Actions
4. 在 `Actions` 页面看到工作流：`Deploy PRISM to GitHub Pages`
5. 点进该工作流，点击 `Run workflow`（首次可手动触发）
6. 等待 `build` 和 `deploy` 绿色成功
7. 回到 `Settings -> Pages`，确认 Published URL
8. DNS 生效后，`Custom domain` 的 DNS 检查应转为成功
9. 最后开启 `Enforce HTTPS`

---

## 6. 日常内容修改与发布流程（本地 -> GitHub）

## 6.1 常见内容文件位置

- 站点全局配置：`content/config.toml`
- 个人简介：`content/bio.md`
- 论文配置：`content/publications.toml`
- 论文数据：`content/publications.bib`
- 博客页配置：`content/blogs.toml`
- 博客正文：`content/blogs.md`
- 杂项页：`content/miscellaneous.toml` / `content/miscellaneous.md`
- 静态资源目录：`public/`
  - 论文预览图：`public/paper_figs/`
  - 论文 PDF：`public/paper_docs/`
  - 幻灯片：`public/slides/`

## 6.2 论文卡片图片绑定逻辑（避免忘记）

在 `content/publications.bib` 的对应条目里写：

```bibtex
preview={your-image.png},
pdfUrl={your-paper.pdf},
```

要求：

- 图片文件放到 `public/paper_figs/your-image.png`
- PDF 文件放到 `public/paper_docs/your-paper.pdf`
- `preview` / `pdfUrl` 只写文件名或相对子路径，不写 `/paper_figs/` 前缀

## 6.3 本地开发与预览

```bash
# 安装依赖（首次或依赖变更时）
npm install

# 本地开发
npm run dev

# 生产构建校验（推荐每次提交前执行）
npm run build
```

## 6.4 提交与推送（标准命令）

```bash
# 查看改动
git status

# 暂存改动
git add .

# 提交（按本次修改内容写清楚）
git commit -m "update website content"

# 推送到主分支
git push origin main
```

推送后：

- GitHub Actions 会自动触发部署
- 到 `Actions` 页查看运行日志
- 绿勾后等待 1~5 分钟刷新站点

---

## 7. 常见问题与排查

### 7.1 Actions 页看到 workflow，但没有运行记录

现象：左侧有工作流名，但中间显示 `0 workflow runs`。

处理：

1. 点左侧具体工作流（不要停在 `All workflows`）
2. 右上角点 `Run workflow`
3. 若没有按钮，检查仓库 Actions 权限：
   - `Settings -> Actions -> General`
   - `Allow all actions and reusable workflows`
   - `Workflow permissions: Read and write permissions`

### 7.2 Pages 提示 DNS check unsuccessful / NotServedByPagesError

原因：域名仍未正确指向 GitHub Pages。

处理：

1. 清理旧 `netlify.app` 解析
2. 按第 4 节改为 GitHub Pages 记录
3. 等待 DNS 传播（通常几分钟到数小时）
4. 在 Pages 页面点 `Check again`

### 7.3 改了 `public/` 目录名后页面报错或资源 404

原因：代码中静态路径未同步。

处理：

1. 全局搜索旧路径：

```bash
rg -n "/papers/|/pdf/|public/papers|public/pdf" src content
```

2. 同步改为当前路径（例如 `/paper_figs/`、`/paper_docs/`）
3. `npm run build` 验证

### 7.4 动态页面报找不到 TOML（ENOENT）

原因：`content/config.toml` 里 `navigation.target` 与实际文件名不一致。

示例：

- `target = "Misc"` 会查找 `content/Misc.toml`
- 若实际是 `content/miscellaneous.toml`，应改成 `target = "miscellaneous"`

---

## 8. 迁移或帮他人搭建时的快速复用模板

1. Fork 或复制项目到新仓库
2. 修改 `content/` 中个人信息与页面内容
3. 配置 `public/CNAME` 为新域名
4. 确认 `.github/workflows/deploy.yml` 触发分支匹配默认分支
5. GitHub Pages Source 选 `GitHub Actions`
6. DNS（Cloudflare）按 GitHub Pages 规范配置
7. 手动触发一次 workflow 并确认成功
8. 等 DNS 与 HTTPS 生效后交付

---

## 9. 建议的日常操作习惯

1. 每次内容更新前先 `git pull origin main`
2. 提交前必须 `npm run build` 一次
3. 提交信息写明确（例如 `update publications`, `add blog post`）
4. 大改目录名时，先全局 `rg` 检查引用再提交
5. 线上异常时优先看 `Actions` 构建日志与 `Pages` 域名状态

---

## 10. 常用命令速查

```bash
# 本地开发
npm run dev

# 构建校验
npm run build

# 查看改动
git status

# 查看哪些文件改了
git diff --name-only

# 提交发布
git add .
git commit -m "update site"
git push origin main

# DNS 解析检查
dig +short bingfan.work A
dig +short www.bingfan.work CNAME
```

