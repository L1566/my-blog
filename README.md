# my-blog

个人博客，基于 [Astro](https://astro.build) 7 的 Bear Blog 主题（MDX + RSS + Sitemap + 本地字体），以纯静态方式部署到 Cloudflare Workers 的静态资源（Static Assets）上。

## 环境要求

- Node.js >= 22.12.0（Cloudflare Workers Builds 默认使用 Node 24，满足要求）

## 本地开发

| 命令 | 作用 |
| :--- | :--- |
| `npm install` | 安装依赖 |
| `npm run dev` | 启动开发服务器 http://localhost:4321 |
| `npm run build` | 生产构建，输出到 `./dist` |
| `npm run preview` | 用 Astro 预览构建产物 |
| `npm run preview:worker` | 构建后用 `wrangler dev` 在本地模拟 Cloudflare 线上行为 |
| `npm run deploy` | 构建并部署到 Cloudflare Workers |
| `npm run astro -- --help` | Astro CLI 帮助 |

## 目录结构

```text
├── public/            # 原样拷贝的静态资源（favicon 等）
├── src/
│   ├── assets/        # 参与构建与优化的图片、字体
│   ├── components/    # BaseHead / Header / Footer 等组件
│   ├── content/blog/  # 博客文章（Markdown / MDX）
│   ├── layouts/       # BlogPost 布局
│   ├── pages/         # 路由：index、about、blog/[...slug]、rss.xml.js、404
│   └── styles/        # 全局样式
├── astro.config.mjs   # Astro 配置（site、MDX、Sitemap、字体）
├── wrangler.jsonc     # Cloudflare Workers 配置（静态资源配置）
└── package.json
```

写文章：在 `src/content/blog/` 新增 `.md`/`.mdx` 文件，frontmatter 字段（title、description、pubDate、heroImage）由 `src/content.config.ts` 校验。

## 部署到 Cloudflare Workers

`wrangler.jsonc` 里的 `assets.directory` 指向 `./dist`，且**没有** `main` 字段——这只上传静态资源，不部署 Worker 脚本。因此本项目不需要 `@astrojs/cloudflare` 适配器。

### 方式一：本地命令行（首次验证可行性）

```sh
npx wrangler login     # 浏览器授权 Cloudflare 账号，只需一次
npm run deploy         # astro build && wrangler deploy
```

首次部署会在账号下创建名为 `my-blog` 的 Worker，并输出访问地址 `https://my-blog.<你的账号子域>.workers.dev`。

### 方式二：Cloudflare Workers Builds（Git 推送自动部署）

1. 把仓库推送到 GitHub（当前远端为 `L1566/my-blog`）。
2. 打开 <https://dash.cloudflare.com> → `Compute` → `Workers & Pages` → `Create application` → `Import a repository`，选择 `L1566/my-blog`。
3. 构建配置：
   - Build command：`npx astro build`
   - Deploy command：`npx wrangler deploy`
4. `Save and Deploy`。此后每次 push 都会自动构建并部署，PR 还会生成预览地址。

### 绑定自定义域名（littlework.top）

`Workers & Pages` → 选择 `my-blog` → `Settings` → `Domains & Routes` → `Add` → `Custom domain` → 填写 `littlework.top`。域名需已托管在同一个 Cloudflare 账号下，DNS 记录由 Cloudflare 自动创建。

## 配置说明

- `astro.config.mjs` 的 `site: 'https://littlework.top'` 决定 canonical、Open Graph、sitemap 与 RSS 里的绝对地址。在 `*.workers.dev` 上做可行性测试时，这些链接仍指向 `littlework.top`，属预期行为；正式域名绑定后即一致。
- `wrangler.jsonc` 的 `not_found_handling: '404-page'` 让 Worker 在找不到资源时返回 `dist/404.html`（由 `src/pages/404.astro` 生成）以及 404 状态码。
- 纯静态站点无法使用 Cloudflare 绑定（KV、D1、R2 等）。将来需要按需渲染或绑定资源时再执行 `npx astro add cloudflare`，并按 Cloudflare 文档给 `wrangler.jsonc` 补上 `main: "@astrojs/cloudflare/entrypoints/server"`。

### lockfile 必须用 CI 的 npm 生成

Cloudflare Workers Builds 用构建镜像自带的 **npm 10.9.2** 执行 `npm ci`，而 Node 24 本地自带 **npm 11**。`sharp` 与 `@astrojs/*` 把 wasm32 兜底二进制声明为可选依赖，npm 11 在本地安装时会把这条链的传递依赖（`@emnapi/core`、`@emnapi/runtime`）从 lockfile 里剪掉，导致 CI 直接失败：

```text
npm error `npm ci` can only install packages when your package.json and package-lock.json ... are in sync.
npm error Missing: @emnapi/runtime@1.11.3 from lock file
npm error Missing: @emnapi/core@1.11.3 from lock file
```

所以**改动依赖后不要把 npm 11 生成的 lockfile 直接提交**，先用与 CI 一致的版本重新生成并自检：

```sh
npx npm@10.9.2 install --package-lock-only   # 重新生成 lockfile
npx npm@10.9.2 ci --dry-run                  # 输出 added N packages 才算通过
```
