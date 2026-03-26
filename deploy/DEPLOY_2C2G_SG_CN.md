# Sub2API 2C2G 新加坡服务器部署说明

本文档面向以下场景：

- 你有一台 `2C2G` 的新加坡 Linux 服务器
- 你希望把 `Sub2API` 直接部署到服务器上
- 你希望避免“本地电脑 -> 远程代理 -> 上游”的额外一跳
- 你需要一份可以照着执行的部署手册

本文默认推荐：

- 使用 Docker Compose 部署
- 使用 `deploy/docker-compose.local.yml`
- 数据落在宿主机本地目录，方便备份和迁移
- 服务器本机直接访问上游，不额外再套一层代理

## 1. 部署方案选择

对于当前项目，推荐优先级如下：

1. 服务器直接使用官方镜像部署
2. 如果你本地代码有修改，再自行打镜像上传服务器

如果你只是想快速上线，优先使用官方镜像。  
如果你本地有未合并改动，或者你就是要部署当前工作区版本，再使用“自定义镜像打包上传”方案。

## 2. 2C2G 机器是否够用

对个人使用、小规模团队、低到中等并发，这个规格是够用的。  
但需要注意，这个项目示例配置偏保守偏大，不能把默认的大连接池参数原样搬到 `2C2G` 机器上。

建议部署目标：

- 管理后台稳定可用
- 少量到中等量 API 请求可正常转发
- 流式请求可用，但不追求高并发

如果后续出现这些情况，建议升级到 `4C4G` 或拆分数据库：

- 并发请求明显增多
- 大量流式请求同时在线
- 日志、监控、统计任务长期占用资源
- PostgreSQL 或 Redis 内存压力持续偏高

## 3. 部署前准备

建议服务器满足以下前提：

- Ubuntu 22.04+/Debian 12+/同类主流 Linux
- 已放行 `22`、`80`、`443`
- 如果暂时不配反代，至少放行 `8080`
- 服务器本机可以稳定访问 OpenAI 上游
- 磁盘建议至少 `30GB`

还建议提前准备：

- 一个域名，后续做 HTTPS
- SSH 密钥登录
- 记录用的密码管理器

## 4. 推荐目录结构

建议在服务器上使用单独目录：

```bash
mkdir -p /opt/sub2api
cd /opt/sub2api
```

后续所有部署文件都放在这里。

## 5. 方式一：使用官方镜像直接部署

这是最省事的方式。

### 5.1 下载部署文件

```bash
cd /opt/sub2api

curl -O https://raw.githubusercontent.com/Wei-Shaw/sub2api/main/deploy/docker-compose.local.yml
curl -O https://raw.githubusercontent.com/Wei-Shaw/sub2api/main/deploy/.env.example

mv docker-compose.local.yml docker-compose.yml
cp .env.example .env
mkdir -p data postgres_data redis_data
```

### 5.2 生成密钥

执行三次，分别保存结果：

```bash
openssl rand -hex 32
```

你至少需要：

- `POSTGRES_PASSWORD`
- `JWT_SECRET`
- `TOTP_ENCRYPTION_KEY`

### 5.3 编辑 `.env`

```bash
nano .env
```

至少修改这些字段：

```env
BIND_HOST=0.0.0.0
SERVER_PORT=8080
SERVER_MODE=release
RUN_MODE=standard
TZ=Asia/Singapore

POSTGRES_USER=sub2api
POSTGRES_PASSWORD=替换成你生成的值
POSTGRES_DB=sub2api

REDIS_PASSWORD=
REDIS_DB=0

ADMIN_EMAIL=admin@example.com
ADMIN_PASSWORD=替换成强密码

JWT_SECRET=替换成你生成的值
TOTP_ENCRYPTION_KEY=替换成你生成的值

DATABASE_MAX_OPEN_CONNS=30
DATABASE_MAX_IDLE_CONNS=10
DATABASE_CONN_MAX_LIFETIME_MINUTES=30
DATABASE_CONN_MAX_IDLE_TIME_MINUTES=5

REDIS_POOL_SIZE=128
REDIS_MIN_IDLE_CONNS=10

GATEWAY_MAX_CONNS_PER_HOST=128
GATEWAY_MAX_IDLE_CONNS=256
GATEWAY_MAX_IDLE_CONNS_PER_HOST=64
```

说明：

- `TZ` 建议用 `Asia/Singapore`
- `ADMIN_PASSWORD` 不建议留空
- `REDIS_PASSWORD` 如果你只在容器内网使用，可以先留空
- 上面这组连接池参数更适合 `2C2G`

## 6. 启动服务

```bash
cd /opt/sub2api
docker compose up -d
docker compose ps
docker compose logs -f sub2api
```

首次启动时，系统会自动完成：

- PostgreSQL 初始化
- Redis 初始化
- 数据库迁移
- 管理员账号创建
- 配置文件生成

如果 `sub2api` 日志里没有持续报错，说明基本已经跑起来了。

## 7. 初次验证

服务启动后，建议先在服务器本机确认健康检查正常：

```bash
curl 127.0.0.1:8080/health
```

建议你至少做这几项验证：

1. 健康检查返回正常
2. 日志里没有持续报错
3. 能进入管理后台
4. 能新增一个上游账号
5. 能发起一次实际请求

如果你已经确认服务器本机可以直连上游，通常不需要在服务器里额外再配本地代理。

## 8. 打包发布流程

如果你本地仓库有修改，或者你要部署当前工作区代码，不建议继续直接拉官方镜像。  
这时建议把发布方式分成三类：

1. 本地构建 Docker 镜像，然后把镜像包上传到服务器
2. 本地打源码包上传到服务器，在服务器上执行 `docker build`
3. 本地构建镜像后推送到制品仓库，服务器直接拉取

推荐优先级：

1. 优先使用“推送到制品仓库再拉取发布”
2. 其次使用“镜像包上传发布”
3. 如果本地 Docker 环境不稳定，或者你希望在服务器完成构建，再使用“源码包上传服务器构建”

### 8.1 发布前的本地检查

无论你走哪条路径，都建议先在本地完成最基本验证：

```bash
cd /path/to/sub2api
git status
```

至少确认：

- 你知道本次要发布的是哪份代码
- 本地改动已经保存
- 如果有关键功能改动，已经做过最基本自测

建议给每次发布一个明确版本名，例如：

```text
sub2api:local-20260326-v1
sub2api:local-20260326-v2
```

这样后续回滚会轻松很多。

### 8.2 方案 A：本地构建镜像并上传服务器

这是最推荐的方案。  
优点是构建结果可控，服务器上只负责加载和运行镜像。

#### 8.2.1 本地构建镜像

```bash
cd /path/to/sub2api
docker build -t sub2api:local-20260326-v1 .
```

#### 8.2.2 本地导出镜像包

```bash
docker save sub2api:local-20260326-v1 | gzip > sub2api-local-20260326-v1.tar.gz
```

#### 8.2.3 上传镜像包到服务器

```bash
scp sub2api-local-20260326-v1.tar.gz root@你的服务器IP:/opt/sub2api/
```

#### 8.2.4 在服务器导入镜像

```bash
cd /opt/sub2api
gunzip -c sub2api-local-20260326-v1.tar.gz | docker load
```

#### 8.2.5 修改 Compose 使用新镜像

编辑：

```bash
nano docker-compose.yml
```

把：

```yaml
image: weishaw/sub2api:latest
```

改成：

```yaml
image: sub2api:local-20260326-v1
```

#### 8.2.6 重启服务

```bash
cd /opt/sub2api
docker compose up -d
docker compose ps
docker compose logs -f sub2api
```

### 8.3 方案 B：本地打源码包，上传服务器后本地构建

这个方案适合：

- 你本地 Docker 环境不好用
- 你希望镜像直接在服务器生成
- 你暂时不想走镜像导出导入

注意：

- 服务器需要已经安装 Docker
- 服务器需要有足够磁盘空间用于构建
- 服务器构建时会拉取基础镜像并安装依赖，耗时通常更长

#### 8.3.1 本地打源码包

在项目根目录执行：

```bash
cd /path/to/sub2api
tar czf sub2api-src-20260326-v1.tar.gz \
  --exclude .git \
  --exclude node_modules \
  --exclude frontend/node_modules \
  --exclude frontend/dist \
  .
```

#### 8.3.2 上传源码包到服务器

```bash
scp sub2api-src-20260326-v1.tar.gz root@你的服务器IP:/opt/sub2api/
```

#### 8.3.3 在服务器解压源码

```bash
cd /opt/sub2api
mkdir -p build-src
tar xzf sub2api-src-20260326-v1.tar.gz -C build-src
```

#### 8.3.4 在服务器构建镜像

```bash
cd /opt/sub2api/build-src
docker build -t sub2api:local-20260326-v1 .
```

#### 8.3.5 修改 Compose 使用新镜像

编辑：

```bash
nano /opt/sub2api/docker-compose.yml
```

把：

```yaml
image: weishaw/sub2api:latest
```

改成：

```yaml
image: sub2api:local-20260326-v1
```

#### 8.3.6 重启服务

```bash
cd /opt/sub2api
docker compose up -d
docker compose logs -f sub2api
```

### 8.4 方案 C：推送到阿里云制品仓库，再由服务器拉取发布

如果你已经有稳定可用的制品仓库，这是最适合长期发布的方案。  
相比 `docker save/scp/docker load`，它更标准，也更适合后续频繁更新。

你提供的登录地址是：

```bash
docker login --username=zhaodonghu_01 crpi-hyxadyfmzekw02ui.cn-zhangjiakou.personal.cr.aliyuncs.com
```

你现在的仓库完整镜像格式是：

```text
crpi-hyxadyfmzekw02ui.cn-zhangjiakou.personal.cr.aliyuncs.com/local-docker-z/sub2api:20260326-v1
```

你提供的拉取示例是：

```bash
docker pull crpi-hyxadyfmzekw02ui.cn-zhangjiakou.personal.cr.aliyuncs.com/local-docker-z/sub2api:[镜像版本号]
```

后文统一使用这个真实仓库路径。

#### 8.4.1 本地登录制品仓库

```bash
docker login --username=zhaodonghu_01 crpi-hyxadyfmzekw02ui.cn-zhangjiakou.personal.cr.aliyuncs.com
```

说明：

- 执行后输入仓库密码或访问令牌
- 不建议把密码直接写进命令行历史

#### 8.4.2 本地构建镜像

```bash
cd /path/to/sub2api
docker build -t sub2api:local-20260326-v1 .
```

#### 8.4.3 给镜像打远程仓库标签

```bash
docker tag sub2api:local-20260326-v1 \
  crpi-hyxadyfmzekw02ui.cn-zhangjiakou.personal.cr.aliyuncs.com/local-docker-z/sub2api:20260326-v1
```

#### 8.4.4 推送到制品仓库

```bash
docker push \
  crpi-hyxadyfmzekw02ui.cn-zhangjiakou.personal.cr.aliyuncs.com/local-docker-z/sub2api:20260326-v1
```

#### 8.4.5 在服务器登录制品仓库

服务器上也要执行一次登录：

```bash
docker login --username=zhaodonghu_01 crpi-hyxadyfmzekw02ui.cn-zhangjiakou.personal.cr.aliyuncs.com
```

#### 8.4.6 修改 Compose 使用仓库镜像

编辑：

```bash
nano /opt/sub2api/docker-compose.yml
```

把：

```yaml
image: weishaw/sub2api:latest
```

改成：

```yaml
image: crpi-hyxadyfmzekw02ui.cn-zhangjiakou.personal.cr.aliyuncs.com/local-docker-z/sub2api:20260326-v1
```

#### 8.4.7 在服务器拉取并启动

如果你想先单独验证某个版本镜像是否已经推送成功，可以先手工执行：

```bash
docker pull crpi-hyxadyfmzekw02ui.cn-zhangjiakou.personal.cr.aliyuncs.com/local-docker-z/sub2api:20260326-v1
```

确认没问题后，再执行：

```bash
cd /opt/sub2api
docker compose pull
docker compose up -d
docker compose ps
docker compose logs -f sub2api
```

#### 8.4.8 这个方案的优点

- 发布流程更标准
- 服务器不需要手工导入镜像包
- 后续升级和回滚都更方便
- 更适合长期维护

### 8.5 推荐的标准发布顺序

建议以后每次都按这个顺序来：

1. 本地修改代码
2. 本地做最基本验证
3. 构建新镜像
4. 使用新标签发布，不覆盖旧标签
5. 服务器更新 `docker-compose.yml`
6. `docker compose up -d`
7. 看日志确认服务正常
8. 再做一次关键功能验证

### 8.6 回滚方式

如果新版本有问题，最简单的回滚方式是：

1. 把 `docker-compose.yml` 里的镜像标签改回上一个版本
2. 再执行：

```bash
cd /opt/sub2api
docker compose up -d
```

所以不建议每次都复用同一个镜像标签。

### 8.7 是否需要重新执行初始化

通常不需要。  
只要你保留了原有的：

- `.env`
- `data/`
- `postgres_data/`
- `redis_data/`

那升级或替换镜像时，数据库和配置都会继续沿用。

## 9. 建议加上反向代理和 HTTPS

正式上线时，不建议长期直接暴露 `8080`。

推荐结构：

- `Nginx/Caddy` 对外监听 `80/443`
- `sub2api` 仅内部监听 `8080`

如果你使用 `Nginx` 并且后续要接 `Codex CLI`，需要特别加：

```nginx
underscores_in_headers on;
```

原因是 `Nginx` 默认会丢弃带下划线的头，例如 `session_id`，这会影响粘性会话。

### 9.1 当前域名方案：`api.nscbm.cn`

如果你当前准备使用的域名是：

```text
api.nscbm.cn
```

建议直接使用 `Caddy` 做反向代理和 HTTPS。

### 9.2 DNS 准备

先在域名服务商后台添加一条 `A` 记录：

- 主机记录：`api`
- 记录值：你的服务器公网 IPv4

在继续之前，先确认解析已经生效。

### 9.3 放行端口

服务器上执行：

```bash
ufw allow 80/tcp
ufw allow 443/tcp
ufw status
```

### 9.4 安装 Caddy

如果服务器是 Ubuntu，可以直接执行：

```bash
apt update
apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt update
apt install -y caddy
```

### 9.5 写入 Caddy 配置

执行：

```bash
cat >/etc/caddy/Caddyfile <<'EOF'
api.nscbm.cn {
    reverse_proxy 127.0.0.1:8080
}
EOF
```

### 9.6 检查并启动 Caddy

执行：

```bash
caddy fmt --overwrite /etc/caddy/Caddyfile
caddy validate --config /etc/caddy/Caddyfile
systemctl enable --now caddy
systemctl restart caddy
systemctl status caddy --no-pager
```

### 9.7 验证 HTTPS

可以直接验证：

```bash
curl -I https://api.nscbm.cn
```

浏览器访问时，直接使用：

```text
https://api.nscbm.cn
```

如果 HTTPS 没起来，优先检查：

- DNS 是否已正确解析到服务器公网 IP
- `80/443` 是否已放行
- `caddy validate` 是否通过
- `systemctl status caddy` 是否正常
- `journalctl -u caddy -n 80 --no-pager` 是否有证书申请错误

## 10. 防火墙和安全建议

部署时至少注意这些：

- 不要把 PostgreSQL 和 Redis 映射到公网
- SSH 建议只用密钥登录
- 为管理员账号设置强密码
- 为 `.env` 做备份，但不要泄露
- 如果使用域名，尽快启用 HTTPS
- 不要把管理后台长期裸露在公网 HTTP 下

建议放行端口：

- `22`：SSH
- `80`：HTTP
- `443`：HTTPS

在反代完成后，可以关闭公网 `8080`。

## 11. 备份与迁移

如果你使用的是 `docker-compose.local.yml` 方案，备份会简单很多。

建议定期备份：

- `/opt/sub2api/.env`
- `/opt/sub2api/data`
- `/opt/sub2api/postgres_data`
- `/opt/sub2api/redis_data`

一个简单的停机打包示例：

```bash
cd /opt/sub2api
docker compose down
cd /opt
tar czf sub2api-backup-$(date +%F).tar.gz sub2api
```

恢复时只要把整个目录还原，再重新执行：

```bash
cd /opt/sub2api
docker compose up -d
```

## 12. 升级方式

如果你使用官方镜像：

```bash
cd /opt/sub2api
docker compose pull
docker compose up -d
```

如果你使用自定义镜像：

- 重新在本地打包
- 上传新镜像
- `docker load`
- 更新 `image` 标签
- 重启容器

升级前建议先做备份。

## 13. 常用排查命令

查看容器状态：

```bash
docker compose ps
```

查看应用日志：

```bash
docker compose logs -f sub2api
```

查看数据库日志：

```bash
docker compose logs -f postgres
```

查看 Redis 日志：

```bash
docker compose logs -f redis
```

重启应用：

```bash
docker compose restart sub2api
```

停止服务：

```bash
docker compose down
```

## 14. 上线前检查清单

- 已确认服务器本机可访问上游
- 已修改 `.env` 中所有关键密码
- 已将连接池参数调小到适合 `2C2G`
- 已成功登录管理后台
- 已完成一次真实上游调用测试
- 已准备反向代理和 HTTPS
- 已确认不会把 PostgreSQL/Redis 暴露到公网
- 已建立最基本的备份流程

## 15. 最终建议

对你当前场景，推荐最终形态是：

1. 把 `Sub2API` 直接部署到新加坡服务器
2. 服务器本机直连上游
3. 前面加 `Nginx` 或 `Caddy` 做 HTTPS
4. 本地客户端直接访问这台服务器

这样链路最短，也最容易维护：

```text
客户端 -> 你的新加坡服务器上的 Sub2API -> 上游
```

不建议长期维持：

```text
客户端 -> 本地电脑 -> 远程代理 -> 上游
```

因为这样多一层中转，延迟更高，排障也更复杂。
