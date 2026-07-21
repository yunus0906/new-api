# 本地开发启动指南

本文面向以前端为主、Go 刚入门的开发者。你的线上环境已经部署完成，因此这里重点只覆盖本地开发、个性化内容修改、Fork 分支管理和日常同步官方更新。

分支管理方案以 [ChatGPT-GitHub Fork.md](./ChatGPT-GitHub%20Fork.md) 为准。当前建议采用两条长期分支：

- `main`：只同步官方仓库，不写自己的业务代码。
- `custom`：自己的本地开发、测试和线上部署分支。

如果某个改动比较大，再临时从 `custom` 拉 `feature/*` 分支，完成后合并回 `custom`。

## 1. 本地需要安装什么

### 必装

- Git：管理 Fork、同步官方更新、切换分支。
- Docker Desktop：本地启动后端、PostgreSQL、Redis。
- Bun：安装和启动前端项目，优先使用 Bun，不使用 npm/yarn/pnpm。

### 建议安装

- Go：如果你只改前端，可以暂时不装；如果需要本机直接跑后端或调试 Go 代码，需要安装。当前 `go.mod` 使用 `go 1.25.1`。
- VS Code / JetBrains IDE：用于编辑代码。
- Make：可选。Windows 没有也没关系，文档里会给出不依赖 `make` 的命令。

### 检查环境

在 PowerShell 中执行：

```powershell
git --version
docker --version
docker compose version
bun --version
go version
```

如果暂时不需要本机跑 Go 后端，`go version` 报错可以先忽略。

## 2. 第一次拉取项目

如果还没有本地代码，先克隆你的 Fork 仓库：

```powershell
git clone <你的 Fork 仓库地址>
cd new-api
```

查看远程仓库：

```powershell
git remote -v
```

通常需要两个 remote：

- `origin`：你的 Fork 仓库。
- `upstream`：官方仓库。

如果还没有 `upstream`，添加官方仓库：

```powershell
git remote add upstream https://github.com/QuantumNous/new-api.git
git fetch upstream
```

## 3. 本地分支准备

### 3.1 保持 main 只同步官方

`main` 不放自己的个性化修改，只用于同步官方：

```powershell
git checkout main
git fetch upstream
git merge upstream/main
git push origin main
```

### 3.2 创建或切换 custom 分支

如果还没有 `custom` 分支：

```powershell
git checkout -b custom main
git push -u origin custom
```

如果已经有 `custom` 分支：

```powershell
git checkout custom
git pull origin custom
```

后续你的个性化内容修改都在 `custom` 或 `feature/*` 分支上做，不直接改 `main`。

## 4. 推荐启动方式：Docker 后端 + 本地前端

这是最适合前端开发的方式：

- 后端、PostgreSQL、Redis 由 Docker 管理。
- 前端在本机用 Bun 启动，热更新更快。
- 前端请求会自动代理到本地 Docker 后端。

### 4.1 启动本地后端环境

确认当前在项目根目录，并且已经切到你的开发分支：

```powershell
git branch --show-current
```

如果不是 `custom` 或你的 `feature/*` 分支，先切过去：

```powershell
git checkout custom
```

启动后端、PostgreSQL、Redis：

```powershell
docker compose -f docker-compose.dev.yml up -d
```

这个命令会启动：

- `new-api-dev`：后端服务，地址是 `http://localhost:3000`
- `new-api-dev-pg`：PostgreSQL 数据库
- `new-api-dev-redis`：Redis 缓存

查看状态：

```powershell
docker compose -f docker-compose.dev.yml ps
```

查看后端日志：

```powershell
docker compose -f docker-compose.dev.yml logs -f new-api
```

### 4.2 安装前端依赖

第一次启动前端前，在项目根目录执行：

```powershell
cd web
bun install --filter ./default
```

如果后续依赖有变化，再重新执行一次。

### 4.3 启动默认前端

继续进入默认前端目录：

```powershell
cd default
bun run dev
```

启动成功后访问终端输出的地址，通常是：

```text
http://localhost:5173
```

`web/default/rsbuild.config.ts` 已配置开发代理，本地前端会把这些请求转发到 `http://localhost:3000`：

- `/api`
- `/mj`
- `/pg`

因此本地前端页面可以直接调用 Docker 后端。

## 5. 第一次打开页面后的初始化

首次访问 `http://localhost:5173` 时，如果本地数据库还没有初始化，会进入初始化流程。

你需要在页面中创建 root 管理员账号：

- 用户名：最多 12 个字符。
- 密码：至少 8 个字符。
- 自用模式 / 演示站点模式：本地开发按需要选择即可。

初始化完成后，用刚创建的 root 账号登录后台。

本地 Docker 数据保存在 Docker volume 里，不会影响线上环境。

## 6. 个性化内容修改建议

你主要改个性化内容时，优先关注默认前端：

```text
web/default/
```

常见修改位置：

- 页面组件：`web/default/src/`
- 静态资源：`web/default/public/`
- 前端路由和页面：`web/default/src/routes/` 或功能目录下的组件
- 多语言文案：`web/default/src/i18n/locales/*.json`

如果新增或修改用户可见文案，需要使用项目的 i18n 方式：

- React 组件里使用 `useTranslation()`。
- 用户可见文本用 `t('English key')`。
- 翻译文件在 `web/default/src/i18n/locales/{lang}.json`。
- 修改后在 `web/default` 执行：

```powershell
bun run i18n:sync
```

项目规则要求保留项目和组织相关受保护标识，不要删除或替换 README、许可证、包名、模块路径、页脚、元数据等位置里的相关项目归属信息。

## 7. 日常开发命令

### 7.1 每天开始开发

```powershell
git checkout custom
git pull origin custom
docker compose -f docker-compose.dev.yml up -d
cd web/default
bun run dev
```

### 7.2 前端检查

在 `web/default` 目录执行：

```powershell
bun run typecheck
bun run lint
bun run build
```

如果只改文案或样式，至少建议跑：

```powershell
bun run typecheck
```

### 7.3 后端容器重建

如果你改了 Go 后端代码，Docker 后端需要重新构建：

```powershell
docker compose -f docker-compose.dev.yml up -d --build new-api
```

只改前端代码时不需要重建后端。

### 7.4 停止本地环境

停止容器但保留本地数据库数据：

```powershell
docker compose -f docker-compose.dev.yml down
```

停止容器并删除本地数据库数据：

```powershell
docker compose -f docker-compose.dev.yml down -v
```

`down -v` 会清空本地开发数据，只在需要重新初始化时使用。

## 8. 同步官方更新

根据 [ChatGPT-GitHub Fork.md](./ChatGPT-GitHub%20Fork.md) 的方案，官方更新先进入 `main`，再合并到 `custom`。

### 8.1 同步 main

```powershell
git checkout main
git fetch upstream
git merge upstream/main
git push origin main
```

### 8.2 合并到 custom

```powershell
git checkout custom
git pull origin custom
git merge main
```

如果没有冲突，推送：

```powershell
git push origin custom
```

如果有冲突，解决冲突后：

```powershell
git status
git add <解决冲突的文件>
git commit
git push origin custom
```

同步官方更新后，建议重新跑本地启动和前端检查：

```powershell
docker compose -f docker-compose.dev.yml up -d --build new-api
cd web/default
bun install --filter ./default
bun run typecheck
bun run build
bun run dev
```

## 9. 开发较大功能时的 feature 分支

如果只是小文案、小样式，可以直接在 `custom` 改。

如果改动较大，从 `custom` 拉功能分支：

```powershell
git checkout custom
git pull origin custom
git checkout -b feature/my-change
```

开发完成后合并回 `custom`：

```powershell
git checkout custom
git merge feature/my-change
git push origin custom
```

确认不再需要功能分支后再删除：

```powershell
git branch -d feature/my-change
git push origin --delete feature/my-change
```

## 10. 可选：本机直接运行 Go 后端

如果你想学习或调试 Go 后端，可以不用 Docker 跑 `new-api` 服务，只用 Docker 启动 PostgreSQL 和 Redis。

### 10.1 准备前端构建产物

后端 `main.go` 使用 `go:embed` 嵌入 `web/default/dist` 和 `web/classic/dist`，所以本机 `go run` 前需要先有这些目录和 `index.html`。

构建两个前端主题：

```powershell
cd web
bun install
cd default
bun run build
cd ../classic
bun run build
```

### 10.2 使用 SQLite 快速启动

回到项目根目录：

```powershell
go run main.go
```

不配置 `SQL_DSN` 时，后端默认使用 SQLite。默认数据库文件是：

```text
one-api.db?_busy_timeout=30000
```

访问：

```text
http://localhost:3000
```

### 10.3 使用 PostgreSQL 和 Redis 启动

先启动依赖：

```powershell
docker compose -f docker-compose.dev.yml up -d postgres redis
```

再设置环境变量并启动 Go：

```powershell
$env:SQL_DSN = "postgresql://root:123456@localhost:5432/new-api"
$env:REDIS_CONN_STRING = "redis://localhost:6379/0"
$env:PORT = "3000"
go run main.go
```

PowerShell 里设置的 `$env:*` 只对当前终端窗口有效，关闭窗口后会失效。

## 11. 本地常见问题

### 11.1 前端页面打不开

确认 `bun run dev` 是否正常运行，并使用终端输出的地址。默认通常是：

```text
http://localhost:5173
```

### 11.2 前端接口 502 或请求失败

确认后端容器是否正常：

```powershell
docker compose -f docker-compose.dev.yml ps
docker compose -f docker-compose.dev.yml logs -f new-api
```

### 11.3 改了 Go 代码但没生效

Docker 后端不会自动重新编译 Go 代码，需要执行：

```powershell
docker compose -f docker-compose.dev.yml up -d --build new-api
```

### 11.4 想重新走初始化流程

如果安装了 `make`：

```powershell
make reset-setup
```

没有 `make` 时，直接重置 Docker volume：

```powershell
docker compose -f docker-compose.dev.yml down -v
docker compose -f docker-compose.dev.yml up -d
```

### 11.5 端口被占用

后端默认占用 `3000`，前端默认通常是 `5173`。

如果后端 `3000` 被占用，可以修改 `docker-compose.dev.yml` 的端口映射，例如：

```yaml
ports:
  - "3001:3000"
```

同时需要让前端代理到新地址。可以在启动前端前设置：

```powershell
$env:VITE_REACT_APP_SERVER_URL = "http://localhost:3001"
cd web/default
bun run dev
```

### 11.6 不小心在 main 上改了代码

先不要提交。可以新建分支把当前改动带过去：

```powershell
git checkout -b feature/move-local-change
```

确认改动在新分支后，再从 `main` 恢复官方同步流程。

## 12. 推荐工作流总结

本地开发：

```powershell
git checkout custom
docker compose -f docker-compose.dev.yml up -d
cd web/default
bun run dev
```

提交个性化修改：

```powershell
git status
git add <文件>
git commit -m "自定义前端内容"
git push origin custom
```

同步官方更新：

```powershell
git checkout main
git fetch upstream
git merge upstream/main
git push origin main

git checkout custom
git merge main
git push origin custom
```

线上部署分支保持使用 `custom`，不要直接部署 `main`。