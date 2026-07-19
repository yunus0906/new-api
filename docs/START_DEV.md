# 本地初始化与启动指南

这份文档面向以前端为主、刚开始接触 Go 的开发者。推荐先用 Docker 启动后端依赖和 Go 服务，再单独启动 `web/default` 前端开发服务器，这样你不需要先理解 Go 的数据库初始化细节，也能完整调试前端页面。

## 1. 准备环境

需要先安装：

- Docker Desktop：用于启动后端、PostgreSQL、Redis。
- Bun：前端依赖安装和脚本运行工具。
- Git：拉取代码。
- Go：只有在你想本机直接运行后端时才需要。当前 `go.mod` 使用 `go 1.25.1`。

可用下面命令检查：

```powershell
docker --version
docker compose version
bun --version
go version
```

## 2. 推荐启动方式：Docker 后端 + 本地前端

### 2.1 启动后端开发环境

在项目根目录执行：

```powershell
docker compose -f docker-compose.dev.yml up -d
```

这个命令会启动：

- `new-api-dev`：后端服务，监听 `http://localhost:3000`
- `new-api-dev-pg`：PostgreSQL 数据库
- `new-api-dev-redis`：Redis 缓存

查看运行状态：

```powershell
docker compose -f docker-compose.dev.yml ps
```

查看后端日志：

```powershell
docker compose -f docker-compose.dev.yml logs -f new-api
```

如果 Go 后端代码有改动，需要重新构建后端容器：

```powershell
docker compose -f docker-compose.dev.yml up -d --build new-api
```

### 2.2 安装前端依赖

在项目根目录执行：

```powershell
cd web
bun install --filter ./default
```

### 2.3 启动默认前端

继续执行：

```powershell
cd default
bun run dev
```

启动后访问终端输出的 Rsbuild 地址，通常是：

```text
http://localhost:5173
```

前端开发服务器会把 `/api`、`/mj`、`/pg` 请求代理到 `http://localhost:3000`，所以本地前端可以直接调用 Docker 中的后端。

## 3. 第一次打开页面后的初始化

首次访问 `http://localhost:5173` 时，如果系统还没有初始化，会进入初始化流程。

你需要在页面中创建 root 管理员账号：

- 用户名：最多 12 个字符。
- 密码：至少 8 个字符。
- 自用模式 / 演示站点模式：按本地开发需要选择即可。

初始化完成后，用刚创建的账号登录后台。

如果想清空初始化状态重新走一遍向导，在项目根目录执行：

```powershell
make reset-setup
```

如果当前环境没有 `make`，可以直接重置 Docker 数据卷：

```powershell
docker compose -f docker-compose.dev.yml down -v
docker compose -f docker-compose.dev.yml up -d
```

注意：`down -v` 会删除开发数据库数据。

## 4. 常用开发命令

在项目根目录：

```powershell
# 启动后端、PostgreSQL、Redis
docker compose -f docker-compose.dev.yml up -d

# 停止开发环境
docker compose -f docker-compose.dev.yml down

# 停止并删除开发数据库数据
docker compose -f docker-compose.dev.yml down -v

# 重新构建后端容器
docker compose -f docker-compose.dev.yml up -d --build new-api
```

在 `web/default` 目录：

```powershell
# 启动前端开发服务器
bun run dev

# 类型检查
bun run typecheck

# 生产构建
bun run build

# i18n 同步
bun run i18n:sync
```

## 5. 可选：本机直接运行 Go 后端

如果你想学习 Go 后端，可以不用 Docker 跑 `new-api` 服务，只保留数据库和 Redis。

### 5.1 创建前端占位构建产物

后端 `main.go` 使用 `go:embed` 嵌入 `web/default/dist` 和 `web/classic/dist`，所以本机 `go run` 前需要先有这些目录和 `index.html`。

最简单的方式是先构建两个前端主题：

```powershell
cd web
bun install
cd default
bun run build
cd ../classic
bun run build
```

如果你只想快速跑后端，也可以手动创建占位文件。

### 5.2 使用 SQLite 启动

回到项目根目录：

```powershell
go run main.go
```

不配置 `SQL_DSN` 时，后端默认使用 SQLite，数据库文件默认是 `one-api.db?_busy_timeout=30000`。

访问：

```text
http://localhost:3000
```

### 5.3 使用 PostgreSQL 和 Redis 启动

先启动依赖：

```powershell
docker compose -f docker-compose.dev.yml up -d postgres redis
```

再设置环境变量并运行后端：

```powershell
$env:SQL_DSN = "postgresql://root:123456@localhost:5432/new-api"
$env:REDIS_CONN_STRING = "redis://localhost:6379/0"
$env:PORT = "3000"
go run main.go
```

## 6. 目录速览

- `main.go`：后端入口，负责初始化配置、数据库、Redis、路由和 HTTP 服务。
- `router/`：路由注册。
- `controller/`：HTTP 请求处理。
- `service/`：业务逻辑。
- `model/`：数据库模型和 GORM 查询。
- `relay/`：AI API 转发和供应商适配。
- `web/default/`：默认前端，React 19 + Rsbuild。
- `web/classic/`：旧版前端，React 18 + Vite。
- `docker-compose.dev.yml`：本地开发推荐使用的 Docker 编排文件。

## 7. 常见问题

### 前端页面打不开

先确认前端服务是否启动成功，并以终端输出的地址为准。默认通常是 `http://localhost:5173`。

### 前端接口 502 或请求失败

确认后端容器是否正常：

```powershell
docker compose -f docker-compose.dev.yml ps
docker compose -f docker-compose.dev.yml logs -f new-api
```

### 改了 Go 代码但没有生效

Docker 方式启动后端时，改 Go 代码需要重新构建：

```powershell
docker compose -f docker-compose.dev.yml up -d --build new-api
```

### 端口被占用

后端默认占用 `3000`，前端默认由 Rsbuild 分配，通常是 `5173`。如果 `3000` 被占用，可以修改 `docker-compose.dev.yml` 的端口映射，或本机运行后端时设置：

```powershell
$env:PORT = "3001"
go run main.go
```

### 忘记管理员账号或想重新初始化

开发环境可以重置数据：

```powershell
docker compose -f docker-compose.dev.yml down -v
docker compose -f docker-compose.dev.yml up -d
```

这会删除本地开发数据库，请只在确认不需要数据时使用。


