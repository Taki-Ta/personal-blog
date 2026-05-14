# AGENT Guide

## 项目作用

本项目是一个基于 Hexo 的个人静态博客站点，用于编写、生成和发布技术文章。

新增或修改博客文章时，必须遵循 [BLOG_WRITING_GUIDE.md](BLOG_WRITING_GUIDE.md) 中的博客写作规范。该规范约束文章语气、写作风格、推荐长度、代码示例风格和 Markdown 结构。

核心用途：

- 使用 Markdown 在 `source/_posts/` 下维护博客文章。
- 使用 Hexo 将文章、分类页、标签页生成到 `public/` 静态目录。
- 使用 `hexo-theme-redefine` 作为博客主题。
- 通过 GitHub Actions 在 `main` 分支更新后自动构建并部署到服务器。

站点配置入口是 `_config.yml`，其中包含站点 URL、文章链接格式、主题配置、分页、分类和标签等 Hexo 配置。

## 主要目录

- `source/_posts/`：博客文章 Markdown 文件。
- `source/categories/`：分类页面。
- `source/tags/`：标签页面。
- `scaffolds/`：Hexo 新建文章、页面、草稿时使用的模板。
- `public/`：Hexo 构建产物目录，已被 `.gitignore` 忽略。
- `themes/`：本地主题目录，目前主题来自 npm 依赖 `hexo-theme-redefine`。
- `.github/workflows/`：GitHub Actions 工作流配置。
- `.ci/`：本地辅助脚本和部署相关文件目录，已被 `.gitignore` 忽略。

## 常用命令

```bash
pnpm install
pnpm run build
pnpm run clean
pnpm run server
```

命令说明：

- `pnpm install`：安装依赖。
- `pnpm run build`：执行 `hexo generate`，生成静态站点到 `public/`。
- `pnpm run clean`：执行 `hexo clean`，清理缓存和构建产物。
- `pnpm run server`：启动本地 Hexo 预览服务。

## CI/CD 流程

CI/CD 配置文件位于 `.github/workflows/deploy-blog.yml`。

触发方式：

- 推送到 `main` 分支时自动触发。
- 在 GitHub Actions 页面手动执行 `workflow_dispatch`。

执行流程：

1. Checkout 仓库代码。
2. 使用 `actions/setup-node@v4` 安装 Node.js 22。
3. 使用 `pnpm/action-setup@v4` 安装 pnpm 10。
4. 执行 `pnpm install --frozen-lockfile`，按 `pnpm-lock.yaml` 安装依赖。
5. 执行 `pnpm exec hexo clean` 清理旧产物。
6. 执行 `pnpm exec hexo generate` 生成静态站点。
7. 使用 `appleboy/scp-action@v0.1.7` 将 `public/*` 上传到服务器目录 `/home/deploy/repos/personal-blog/public`。

部署所需的服务器连接信息来自 GitHub Secrets：

- `BLOG_SERVER_HOST`：服务器地址。
- `BLOG_SERVER_USER`：SSH 用户名。
- `BLOG_DEPLOY_SSH_KEY`：部署用 SSH 私钥。
- `BLOG_SERVER_PORT`：SSH 端口。

工作流设置了并发组 `deploy-blog-production`，如果同一时间有新的部署任务启动，会取消正在运行的旧任务，避免多个部署互相覆盖。

## 注意事项

- `public/` 是构建产物，不应提交到 Git。
- `db.json` 是 Hexo 缓存文件，不应提交到 Git。
- `.ci/` 已被忽略，如果其中存放私钥或本地脚本，应避免误提交。
- 修改博客配置后建议先执行 `pnpm run build`，确认 Hexo 可以正常生成站点。
