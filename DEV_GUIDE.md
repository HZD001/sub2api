# sub2api 项目开发指南

> 本文档记录项目环境配置、常见坑点和注意事项，供 Claude Code 和团队成员参考。

## 一、项目基本信息

| 项目 | 说明 |
|------|------|
| **上游仓库** | Wei-Shaw/sub2api |
| **Fork 仓库** | bayma888/sub2api-bmai |
| **技术栈** | Go 后端 (Ent ORM + Gin) + Vue3 前端 (pnpm) |
| **数据库** | PostgreSQL 16 + Redis |
| **包管理** | 后端: go modules, 前端: **pnpm**（不是 npm） |

## 二、本地环境配置

### PostgreSQL 16 (Windows 服务)

| 配置项 | 值 |
|--------|-----|
| 端口 | 5432 |
| psql 路径 | `C:\Program Files\PostgreSQL\16\bin\psql.exe` |
| pg_hba.conf | `C:\Program Files\PostgreSQL\16\data\pg_hba.conf` |
| 数据库凭据 | user=`sub2api`, password=`sub2api`, dbname=`sub2api` |
| 超级用户 | user=`postgres`, password=`postgres` |

### Redis

| 配置项 | 值 |
|--------|-----|
| 端口 | 6379 |
| 密码 | 无 |

### 开发工具

```bash
# golangci-lint v2.7
go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.7

# pnpm (前端包管理)
npm install -g pnpm
```

## 三、本地从 0 到 1 联调流程

这一节只解决一件事：把项目在本地稳定跑起来，并且知道出问题先查哪里。

### 3.1 推荐的两种调试方式

#### 方式 A：源码分离联调（推荐给日常开发）

适合前端、后端分别调试，改代码后反馈最快。

```bash
# 1. 启动 PostgreSQL / Redis
# 确保本机已有：
# PostgreSQL: 127.0.0.1:5432
# Redis: 127.0.0.1:6379

# 2. 启动后端
cd backend
go run ./cmd/server/

# 3. 启动前端
cd ../frontend
pnpm install
pnpm dev
```

访问：

- 前端开发页：`http://localhost:3000`
- 后端接口：`http://localhost:8080`

说明：

- `go run ./cmd/server/` 不会内嵌前端页面，前端必须单独执行 `pnpm dev`
- 前端开发服务器会把 `/api`、`/v1`、`/setup` 代理到后端
- 如果是首次启动，浏览器会自动进入 `/setup`

#### 方式 B：Docker 一体化联调

适合快速验证服务能否整体跑通。

```bash
cd deploy
cp .env.example .env

# 至少补一个值
# POSTGRES_PASSWORD=你的密码

docker compose -f docker-compose.dev.yml up --build
```

说明：

- 这种方式会同时拉起 `sub2api + postgres + redis`
- 镜像构建时会把前端打包进后端，不需要单独启动 `pnpm dev`

### 3.2 第一次本地启动建议顺序

#### 第 1 步：确认数据库和 Redis 已启动

最小检查：

```bash
# PostgreSQL
psql -U sub2api -h 127.0.0.1 -d sub2api

# Redis
redis-cli -h 127.0.0.1 -p 6379 ping
```

如果 PostgreSQL 报：

```text
FATAL: role "sub2api" does not exist
```

先用超级用户创建账号和数据库：

```sql
CREATE ROLE sub2api WITH LOGIN PASSWORD 'sub2api';
CREATE DATABASE sub2api OWNER sub2api;
GRANT ALL PRIVILEGES ON DATABASE sub2api TO sub2api;
```

#### 第 2 步：先起后端，再起前端

推荐固定约定：

- 后端：`8080`
- 前端：`3000`
- PostgreSQL：`5432`
- Redis：`6379`

#### 第 3 步：首次初始化填写建议

如果走 `/setup`：

- Database Host：`127.0.0.1`
- Database Port：`5432`
- Database User：`sub2api`
- Database Password：`sub2api`
- Database Name：`sub2api`
- SSL Mode：`disable`
- Redis Host：`127.0.0.1`
- Redis Port：`6379`
- Redis Password：留空
- Redis DB：`0`
- Redis TLS：关闭
- Admin Email：自己设置
- Admin Password：自己设置，至少 8 位
- Server Host：`0.0.0.0`
- Server Port：`8080`
- Server Mode：开发环境建议 `debug`

#### 第 4 步：初始化完成后，手动重启后端

这一条非常重要。

源码模式下，setup 完成后页面会提示“服务自动重启”，但本地 `go run` 场景不能依赖这个行为。最稳妥的做法是：

```bash
# 结束当前 backend 的 go run
# 然后重新执行
cd backend
go run ./cmd/server/
```

重启后再登录，不要直接依赖 setup 页面跳转结果。

### 3.3 初始化完成后必须人工检查的配置

重点看 `backend/config.yaml`：

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

如果这里写成了：

- `server.port: 3000`
- `database.password: ""`
- `database.host: localhost`

那后续大概率会出现端口冲突、数据库连接失败、登录 404 / 接口 500 等问题。

### 3.4 本地联调时最常用的自检命令

```bash
# 看端口被谁占用
lsof -nP -iTCP:3000 -sTCP:LISTEN
lsof -nP -iTCP:8080 -sTCP:LISTEN
lsof -nP -iTCP:5432 -sTCP:LISTEN
lsof -nP -iTCP:6379 -sTCP:LISTEN

# 看后端健康检查
curl -I http://127.0.0.1:8080/health

# 看 setup 状态
curl http://127.0.0.1:8080/setup/status
```

判断逻辑：

- 前端页面能开，但接口全 500：先看后端是否真的还活着
- 登录 404：大概率还停留在 setup 模式，或者后端根本没启动
- 前端跑到 `3001`：大概率 `3000` 已经被别的进程占用了

## 四、CI/CD 流水线

### GitHub Actions Workflows

| Workflow | 触发条件 | 检查内容 |
|----------|----------|----------|
| **backend-ci.yml** | push, pull_request | 单元测试 + 集成测试 + golangci-lint v2.7 |
| **security-scan.yml** | push, pull_request, 每周一 | govulncheck + gosec + pnpm audit |
| **release.yml** | tag `v*` | 构建发布（PR 不触发） |

### CI 要求

- Go 版本必须是 **1.25.7**
- 前端使用 `pnpm install --frozen-lockfile`，必须提交 `pnpm-lock.yaml`

### 本地测试命令

```bash
# 后端单元测试
cd backend && go test -tags=unit ./...

# 后端集成测试
cd backend && go test -tags=integration ./...

# 代码质量检查
cd backend && golangci-lint run ./...

# 前端依赖安装（必须用 pnpm）
cd frontend && pnpm install
```

## 五、常见坑点 & 解决方案

### 坑 1：pnpm-lock.yaml 必须同步提交

**问题**：`package.json` 新增依赖后，CI 的 `pnpm install --frozen-lockfile` 失败。

**原因**：上游 CI 使用 pnpm，lock 文件不同步会报错。

**解决**：
```bash
cd frontend
pnpm install  # 更新 pnpm-lock.yaml
git add pnpm-lock.yaml
git commit -m "chore: update pnpm-lock.yaml"
```

---

### 坑 2：npm 和 pnpm 的 node_modules 冲突

**问题**：之前用 npm 装过 `node_modules`，pnpm install 报 `EPERM` 错误。

**解决**：
```bash
cd frontend
rm -rf node_modules  # 或 PowerShell: Remove-Item -Recurse -Force node_modules
pnpm install
```

---

### 坑 3：PowerShell 中 bcrypt hash 的 `$` 被转义

**问题**：bcrypt hash 格式如 `$2a$10$xxx...`，PowerShell 把 `$2a` 当变量解析，导致数据丢失。

**解决**：将 SQL 写入文件，用 `psql -f` 执行：
```bash
# 错误示范（PowerShell 会吃掉 $）
psql -c "INSERT INTO users ... VALUES ('$2a$10$...')"

# 正确做法
echo "INSERT INTO users ... VALUES ('\$2a\$10\$...')" > temp.sql
psql -U sub2api -h 127.0.0.1 -d sub2api -f temp.sql
```

---

### 坑 4：psql 不支持中文路径

**问题**：`psql -f "D:\中文路径\file.sql"` 报错找不到文件。

**解决**：复制到纯英文路径再执行：
```bash
cp "D:\中文路径\file.sql" "C:\temp.sql"
psql -f "C:\temp.sql"
```

---

### 坑 5：PostgreSQL 密码重置流程

**场景**：忘记 PostgreSQL 密码。

**步骤**：
1. 修改 `C:\Program Files\PostgreSQL\16\data\pg_hba.conf`
   ```
   # 将 scram-sha-256 改为 trust
   host    all    all    127.0.0.1/32    trust
   ```
2. 重启 PostgreSQL 服务
   ```powershell
   Restart-Service postgresql-x64-16
   ```
3. 无密码登录并重置
   ```bash
   psql -U postgres -h 127.0.0.1
   ALTER USER sub2api WITH PASSWORD 'sub2api';
   ALTER USER postgres WITH PASSWORD 'postgres';
   ```
4. 改回 `scram-sha-256` 并重启

---

### 坑 6：Go interface 新增方法后 test stub 必须补全

**问题**：给 interface 新增方法后，编译报错 `does not implement interface (missing method XXX)`。

**原因**：所有测试文件中实现该 interface 的 stub/mock 都必须补上新方法。

**解决**：
```bash
# 搜索所有实现该 interface 的 struct
cd backend
grep -r "type.*Stub.*struct" internal/
grep -r "type.*Mock.*struct" internal/

# 逐一补全新方法
```

---

### 坑 7：Windows 上 psql 连 localhost 的 IPv6 问题

**问题**：psql 连 `localhost` 先尝试 IPv6 (::1)，可能报错后再回退 IPv4。

**建议**：直接用 `127.0.0.1` 代替 `localhost`。

---

### 坑 8：Windows 没有 make 命令

**问题**：CI 里用 `make test-unit`，本地 Windows 没有 make。

**解决**：直接用 Makefile 里的原始命令：
```bash
# 代替 make test-unit
go test -tags=unit ./...

# 代替 make test-integration
go test -tags=integration ./...
```

---

### 坑 9：Ent Schema 修改后必须重新生成

**问题**：修改 `ent/schema/*.go` 后，代码不生效。

**解决**：
```bash
cd backend
go generate ./ent  # 重新生成 ent 代码
git add ent/       # 生成的文件也要提交
```

---

### 坑 10：前端测试看似正常，但后端调用失败（模型映射被批量误改）

**典型现象**：
- 前端按钮点测看起来正常；
- 实际通过 API/客户端调用时返回 `Service temporarily unavailable` 或提示无可用账号；
- 常见于 OpenAI 账号（例如 Codex 模型）在批量修改后突然不可用。

**根因**：
- OpenAI 账号编辑页默认不显式展示映射规则，容易让人误以为“没映射也没关系”；
- 但在**批量修改同时选中不同平台账号**（OpenAI + Antigravity/Gemini）时，模型白名单/映射可能被跨平台策略覆盖；
- 结果是 OpenAI 账号的关键模型映射丢失或被改坏，后端选不到可用账号。

**修复方案（按优先级）**：
1. **快速修复（推荐）**：在批量修改中补回正确的透传映射（例如 `gpt-5.3-codex -> gpt-5.3-codex-spark`）。
2. **彻底重建**：删除并重新添加全部相关账号（最稳但成本高）。

**关键经验**：
- 如果某模型已被软件内置默认映射覆盖，通常不需要额外再加透传；
- 但当上游模型更新快于本仓库默认映射时，**手动批量添加透传映射**是最简单、最低风险的临时兜底方案；
- 批量操作前尽量按平台分组，不要混选不同平台账号。

---

### 坑 11：PR 提交前检查清单

提交 PR 前务必本地验证：

- [ ] `go test -tags=unit ./...` 通过
- [ ] `go test -tags=integration ./...` 通过
- [ ] `golangci-lint run ./...` 无新增问题
- [ ] `pnpm-lock.yaml` 已同步（如果改了 package.json）
- [ ] 所有 test stub 补全新接口方法（如果改了 interface）
- [ ] Ent 生成的代码已提交（如果改了 schema）

---

### 坑 12：README 里的 demo 账号不是你本地初始化账号

**问题**：看到 README 里的 `admin@sub2api.org / admin123`，误以为本地也能直接登录。

**结论**：这是官方演示站账号，不会自动创建到你的本地数据库。

**正确做法**：

- 源码模式：管理员账号是在 `/setup` 里自己创建
- Docker `AUTO_SETUP`：管理员账号取决于 `.env` 中的 `ADMIN_EMAIL / ADMIN_PASSWORD`

---

### 坑 13：`role "sub2api" does not exist`

**问题**：后端或 `psql` 连接 PostgreSQL 时提示：

```text
FATAL: role "sub2api" does not exist
```

**原因**：PostgreSQL 服务虽然起来了，但项目使用的数据库用户还没建。

**解决**：

```sql
CREATE ROLE sub2api WITH LOGIN PASSWORD 'sub2api';
CREATE DATABASE sub2api OWNER sub2api;
GRANT ALL PRIVILEGES ON DATABASE sub2api TO sub2api;
```

---

### 坑 14：源码模式 setup 完成后，不会像生产那样自动重启

**问题**：setup 页面显示安装成功，但点击登录后接口 404。

**根因**：

- 本地是 `go run ./cmd/server/`
- setup 完成后代码会尝试“自动重启服务”
- 该重启逻辑依赖 Linux + systemd，本地开发场景通常不会真正切换到正常服务模式

**解决**：

```bash
# 手动结束 backend 进程
# 然后重新启动
cd backend
go run ./cmd/server/
```

---

### 坑 15：setup 页面会把浏览器当前端口写进后端配置

**问题**：初始化后 `backend/config.yaml` 里 `server.port` 变成了 `3000`。

**后果**：

- 后端和前端开发服务器争抢同一个端口
- Vite 自动切到 `3001`
- 接口代理和登录链路开始混乱

**解决**：

- setup 时明确把 Server Port 填成 `8080`
- 初始化后手动检查 `backend/config.yaml`

---

### 坑 16：前端页面能打开，但接口全是 500，不一定是业务逻辑报错

**问题**：页面能访问，但所有接口都报 500。

**根因**：

- 很多时候不是后端代码真的返回了 500
- 而是前端开发服务器代理不到后端，例如：
  - 后端没启动
  - 后端端口写错
  - 后端启动即退出

**排查顺序**：

```bash
lsof -nP -iTCP:8080 -sTCP:LISTEN
curl -I http://127.0.0.1:8080/health
```

---

### 坑 17：清理 `3000` 端口时，可能误杀后端

**问题**：为了让前端回到 `3000`，直接把占用 `3000` 的进程 kill 掉，结果接口全部挂了。

**原因**：如果 setup 时把后端配置成了 `3000`，那占用 `3000` 的可能是后端，不是前端。

**正确做法**：

```bash
lsof -nP -iTCP:3000 -sTCP:LISTEN
```

先看清楚进程名和 PID，再决定要不要杀。

---

### 坑 18：`localhost` 能用，但本地联调仍建议优先写 `127.0.0.1`

**问题**：某些环境下 `localhost` 会优先走 IPv6（如 `::1`），导致连接行为和预期不一致。

**建议**：

- PostgreSQL / Redis / 后端联调统一使用 `127.0.0.1`
- 特别是配置文件、脚本、初始化表单，优先用 `127.0.0.1`

---

### 坑 19：初始化成功不代表 `config.yaml` 一定正确

**问题**：setup 表面成功，但后端重启后仍然连不上数据库或端口冲突。

**原因**：setup 成功只代表安装流程通过，不代表最终写入的配置完全符合你的本地联调预期。

**建议**：初始化后人工检查以下字段：

- `backend/config.yaml -> server.port`
- `backend/config.yaml -> database.password`
- `backend/config.yaml -> database.host`
- `backend/config.yaml -> redis.host`

## 六、常用命令速查

### 数据库操作

```bash
# 连接数据库
psql -U sub2api -h 127.0.0.1 -d sub2api

# 查看所有用户
psql -U postgres -h 127.0.0.1 -c "\du"

# 查看所有数据库
psql -U postgres -h 127.0.0.1 -c "\l"

# 执行 SQL 文件
psql -U sub2api -h 127.0.0.1 -d sub2api -f migration.sql
```

### PostgreSQL / Redis 启停

```bash
# macOS (Homebrew)
brew services start postgresql@16
brew services stop postgresql@16
brew services start redis
brew services stop redis

# Docker Compose
cd deploy
docker compose -f docker-compose.dev.yml up -d postgres redis
docker compose -f docker-compose.dev.yml stop postgres redis
```

### 端口排查

```bash
lsof -nP -iTCP:3000 -sTCP:LISTEN
lsof -nP -iTCP:3001 -sTCP:LISTEN
lsof -nP -iTCP:8080 -sTCP:LISTEN
lsof -nP -iTCP:5432 -sTCP:LISTEN
lsof -nP -iTCP:6379 -sTCP:LISTEN
```

### Git 操作

```bash
# 同步上游
git fetch upstream
git checkout main
git merge upstream/main
git push origin main

# 创建功能分支
git checkout -b feature/xxx

# Rebase 到最新 main
git fetch upstream
git rebase upstream/main
```

### 前端操作

```bash
# 安装依赖（必须用 pnpm）
cd frontend
pnpm install

# 开发服务器
pnpm dev

# 构建
pnpm build
```

### 后端操作

```bash
# 运行服务器
cd backend
go run ./cmd/server/

# 生成 Ent 代码
go generate ./ent

# 运行测试
go test -tags=unit ./...
go test -tags=integration ./...

# Lint 检查
golangci-lint run ./...
```

## 七、项目结构速览

```
sub2api-bmai/
├── backend/
│   ├── cmd/server/          # 主程序入口
│   ├── ent/                 # Ent ORM 生成代码
│   │   └── schema/          # 数据库 Schema 定义
│   ├── internal/
│   │   ├── handler/         # HTTP 处理器
│   │   ├── service/         # 业务逻辑
│   │   ├── repository/      # 数据访问层
│   │   └── server/          # 服务器配置
│   ├── migrations/          # 数据库迁移脚本
│   └── config.yaml          # 配置文件
├── frontend/
│   ├── src/
│   │   ├── api/             # API 调用
│   │   ├── components/      # Vue 组件
│   │   ├── views/           # 页面视图
│   │   ├── types/           # TypeScript 类型
│   │   └── i18n/            # 国际化
│   ├── package.json         # 依赖配置
│   └── pnpm-lock.yaml       # pnpm 锁文件（必须提交）
└── .claude/
    └── CLAUDE.md            # 本文档
```

## 八、参考资源

- [上游仓库](https://github.com/Wei-Shaw/sub2api)
- [Ent 文档](https://entgo.io/docs/getting-started)
- [Vue3 文档](https://vuejs.org/)
- [pnpm 文档](https://pnpm.io/)
