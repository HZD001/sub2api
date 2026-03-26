# Sub2API 本地开发启动文档

> 这份文档只解决一件事：在本地把项目稳定跑起来，并知道第一次启动和日常启动分别该做什么。

## 1. 本地开发推荐端口

- 前端：`3000`
- 后端：`8080`
- PostgreSQL：`5432`
- Redis：`6379`

推荐固定这样约定，不要把后端也跑到 `3000`。

## 2. 本地开发推荐方式

### 方式 A：源码分离联调

适合日常开发。

- 前端单独启动 `pnpm dev`
- 后端单独启动 `go run ./cmd/server/`
- PostgreSQL / Redis 用本机服务或 Docker 单独起

### 方式 B：Docker 一体化

适合快速验证整体服务是否可运行。

- `sub2api + postgres + redis` 一起拉起
- 前端已打包进后端，不需要再单独运行 `pnpm dev`

## 3. 第一次启动前要准备的东西

### 3.1 开发工具

```bash
# 安装 pnpm
npm install -g pnpm

# 可选：安装 golangci-lint
go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.7
```

### 3.2 PostgreSQL

项目本地联调默认使用：

- host: `127.0.0.1`
- port: `5432`
- user: `sub2api`
- password: `sub2api`
- dbname: `sub2api`

如果数据库用户和库还没建，先执行：

```sql
CREATE ROLE sub2api WITH LOGIN PASSWORD 'sub2api';
CREATE DATABASE sub2api OWNER sub2api;
GRANT ALL PRIVILEGES ON DATABASE sub2api TO sub2api;
```

### 3.3 Redis

项目本地联调默认使用：

- host: `127.0.0.1`
- port: `6379`
- password: 空
- db: `0`

## 4. 第一次启动：源码分离联调

### 4.1 启动 PostgreSQL / Redis

如果你用 Homebrew：

```bash
brew services start postgresql@16
brew services start redis
```

如果你用 Docker Compose：

```bash
cd deploy
docker compose -f docker-compose.dev.yml up -d postgres redis
```

### 4.2 检查 PostgreSQL / Redis 是否可用

```bash
# PostgreSQL
psql -U sub2api -h 127.0.0.1 -d sub2api

# Redis
redis-cli -h 127.0.0.1 -p 6379 ping
```

如果 Redis 正常，会返回：

```text
PONG
```

### 4.3 启动后端

```bash
cd backend
go run ./cmd/server/
```

首次启动时，因为还没有安装完成，会进入 setup 模式。

### 4.4 启动前端

```bash
cd frontend
pnpm install
pnpm dev
```

默认访问：

- 前端页面：`http://localhost:3000`
- 后端接口：`http://localhost:8080`

### 4.5 首次 setup 页面填写建议

浏览器打开前端后，会自动进入 `/setup`。

按下面填写：

#### Database

- Host: `127.0.0.1`
- Port: `5432`
- Username: `sub2api`
- Password: `sub2api`
- Database Name: `sub2api`
- SSL Mode: `disable`

#### Redis

- Host: `127.0.0.1`
- Port: `6379`
- Password: 留空
- DB: `0`
- Enable TLS: 关闭

#### Admin

- Email: 自己设置
- Password: 自己设置，至少 8 位
- Confirm Password: 与密码一致

#### Server

- Host: `0.0.0.0`
- Port: `8080`
- Mode: `debug`

### 4.6 setup 完成后必须做的事

源码模式下，不要依赖 setup 页面提示的“自动重启”。

第一次安装成功后，手动重启后端：

```bash
cd backend
go run ./cmd/server/
```

如果之前那个后端进程还在，先结束它，再重新执行。

### 4.7 检查 `backend/config.yaml`

第一次 setup 完成后，至少确认这几项：

```yaml
server:
  host: 0.0.0.0
  port: 8080
  mode: debug

database:
  host: 127.0.0.1
  port: 5432
  user: sub2api
  password: sub2api
  dbname: sub2api
  sslmode: disable

redis:
  host: 127.0.0.1
  port: 6379
  password: ""
  db: 0
  enable_tls: false
```

尤其注意不要写成：

- `server.port: 3000`
- `database.password: ""`
- `database.host: localhost`

## 5. 日常启动：源码分离联调

完成首次 setup 之后，日常启动就是下面这套。

### 5.1 启动 PostgreSQL / Redis

```bash
brew services start postgresql@16
brew services start redis
```

或者：

```bash
cd deploy
docker compose -f docker-compose.dev.yml up -d postgres redis
```

### 5.2 启动后端

```bash
cd backend
go run ./cmd/server/
```

### 5.3 启动前端

```bash
cd frontend
pnpm dev
```

## 6. 一体化启动：Docker 开发模式

适合快速验证，不适合前后端分别热调。

### 6.1 第一次启动

```bash
cd deploy
cp .env.example .env
```

然后编辑 `.env`，至少设置：

```bash
POSTGRES_PASSWORD=your_password
```

启动：

```bash
docker compose -f docker-compose.dev.yml up --build
```

### 6.2 日常启动

```bash
cd deploy
docker compose -f docker-compose.dev.yml up -d
```

### 6.3 停止

```bash
cd deploy
docker compose -f docker-compose.dev.yml stop
```

## 7. 停止各个服务

### 7.1 停止前端

在运行 `pnpm dev` 的终端直接 `Ctrl+C`。

### 7.2 停止后端

在运行 `go run ./cmd/server/` 的终端直接 `Ctrl+C`。

### 7.3 停止 PostgreSQL / Redis

Homebrew：

```bash
brew services stop postgresql@16
brew services stop redis
```

Docker Compose：

```bash
cd deploy
docker compose -f docker-compose.dev.yml stop postgres redis
```

## 8. 常用排查命令

### 8.1 看端口被谁占用

```bash
lsof -nP -iTCP:3000 -sTCP:LISTEN
lsof -nP -iTCP:3001 -sTCP:LISTEN
lsof -nP -iTCP:8080 -sTCP:LISTEN
lsof -nP -iTCP:5432 -sTCP:LISTEN
lsof -nP -iTCP:6379 -sTCP:LISTEN
```

### 8.2 看后端是否活着

```bash
curl -I http://127.0.0.1:8080/health
```

### 8.3 看 setup 状态

```bash
curl http://127.0.0.1:8080/setup/status
```

### 8.4 看前端为什么跑到了 3001

通常是 `3000` 已被占用：

```bash
lsof -nP -iTCP:3000 -sTCP:LISTEN
```

## 9. 最常见的坑

### 坑 1：README 里的 demo 账号不能直接登录本地

- `admin@sub2api.org / admin123` 是演示环境账号
- 源码本地联调时，管理员账号是在 `/setup` 里自己创建

### 坑 2：setup 成功后，源码模式下要手动重启后端

- 不要依赖 setup 页的自动重启提示
- 最稳妥的方法是手动结束后端，再重新执行 `go run ./cmd/server/`

### 坑 3：后端端口不要填成 3000

- `3000` 应该留给前端开发服务器
- 后端固定用 `8080`

### 坑 4：接口全 500，先看后端是不是根本没起来

先执行：

```bash
lsof -nP -iTCP:8080 -sTCP:LISTEN
curl -I http://127.0.0.1:8080/health
```

### 坑 5：`role "sub2api" does not exist`

说明 PostgreSQL 用户还没建，不是后端代码问题。

### 坑 6：本地联调尽量统一写 `127.0.0.1`

- 避免 `localhost` 在某些环境优先解析到 IPv6
- 特别是 PostgreSQL / Redis / 后端配置

### 坑 7：`http://localhost:3000/responses` 返回 404

- `3000` 是前端 Vite 开发服务器，不是后端 API 服务
- 本地开发时，前端只会代理 `/api`、`/v1`、`/setup`
- 所以 `http://localhost:3000/responses` 会 404

正确调用方式：

- 直接打后端：`http://localhost:8080/v1/responses`
- 或走前端代理：`http://localhost:3000/v1/responses`

如果你用的是 OpenAI SDK / Codex 这类会自动在 `baseURL` 后拼 `/responses` 的客户端：

- `baseURL` 应该写成 `http://localhost:8080/v1`
- 不要写成 `http://localhost:3000`

### 坑 8：`HTTP 403 {"code":"INSUFFICIENT_BALANCE","message":"Insufficient account balance"}`

- 这通常不是上游 OpenAI 返回的
- 而是本地项目先把请求拦下来了
- 说明当前 API Key 对应用户的本地余额 `<= 0`

最快确认方式：

```bash
curl http://localhost:8080/v1/usage \
  -H "Authorization: Bearer sk-你的key"
```

这里的路径一定要区分清楚：

- `GET /v1/usage`：sub2api 本地网关接口，给 API Key 查本地余额、额度和限流
- `GET /api/v1/usage`：后台管理接口，走登录态 / JWT，不是同一个入口
- Anthropic OAuth 上游 usage 在本项目里使用 `https://api.anthropic.com/api/oauth/usage`，不是 `/v1/usage`

所以如果你在测上游服务，或者误把后台接口当成网关接口来测，`/v1/usage` 很可能就是 404。

如果返回里的 `balance` / `remaining` 是 `0`，就说明是本地余额不足。

处理方式：

- 去后台给该用户加余额
- 或把后台设置里的 `default_balance` 调大，避免新用户默认余额为 `0`
- 纯本地联调时，也可以把后端切到 `simple` 模式跳过余额检查，但只建议开发环境使用

## 10. 推荐阅读

- [项目开发指南](../DEV_GUIDE.md)
- [项目说明（中文）](../README_CN.md)
- [部署说明](../deploy/README.md)
