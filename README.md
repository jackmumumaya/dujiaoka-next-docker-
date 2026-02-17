# Dujiao-Next Docker 部署指南

> 本文档将从零开始，手把手教你在 Linux 服务器上使用 Docker 部署 Dujiao-Next 完整项目。

---

## 📋 目录

1. [项目架构说明](#1-项目架构说明)
2. [服务器要求](#2-服务器要求)
3. [安装 Docker](#3-安装-docker)
4. [准备部署文件](#4-准备部署文件)
5. [修改配置文件](#5-修改配置文件)
6. [一键部署](#6-一键部署)
7. [手动部署（逐步操作）](#7-手动部署逐步操作)
8. [配置反向代理（域名 + HTTPS）](#8-配置反向代理域名--https)
9. [日常运维命令](#9-日常运维命令)
10. [数据备份与恢复](#10-数据备份与恢复)
11. [升级指南](#11-升级指南)
12. [支付网关集成](#12-支付网关集成)
13. [常见问题排查](#13-常见问题排查)

---

## 1. 项目架构说明

Dujiao-Next 由以下 **4 个服务** 组成：

```
┌─────────────────────────────────────────────────────┐
│                    用户浏览器                         │
│                                                     │
│         ┌──────────┐       ┌──────────┐             │
│         │ 用户前端  │       │ 管理后台  │              │
│         │ :9092    │       │ :9091    │              │
│         └────┬─────┘       └────┬─────┘             │
│              │   /api /uploads  │                    │
│              └────────┬─────────┘                    │
│                       ▼                              │
│              ┌──────────────┐                        │
│              │  后端 API     │  (仅内部访问)           │
│              │  Go :9090    │                        │
│              └───┬──────┬───┘                        │
│                  │      │                            │
│           ┌──────▼┐  ┌──▼──────┐                    │
│           │ Redis │  │ SQLite  │                    │
│           │ :6379 │  │  数据库  │                    │
│           └───────┘  └─────────┘                    │
└─────────────────────────────────────────────────────┘
```

| 服务     | 技术栈          | 容器名       | 端口 | 对外暴露 | 说明                       |
| -------- | --------------- | ------------ | ---- | -------- | -------------------------- |
| 后端 API | Go (Gin + GORM) | dujiao-api   | 9090 | ❌ 否    | 核心业务逻辑（仅内部访问） |
| 管理后台 | Vue 3 + Vite    | dujiao-admin | 9091 | ✅ 是    | 管理员操作界面             |
| 用户前端 | Vue 3 + Vite    | dujiao-user  | 9092 | ✅ 是    | 用户访问界面               |
| Redis    | Redis 7         | dujiao-redis | 6379 | ❌ 否    | 缓存 & 消息队列（仅内部）  |

---

## 2. 服务器要求

### 最低配置

| 项目         | 要求                                                    |
| ------------ | ------------------------------------------------------- |
| **操作系统** | Ubuntu 20.04+ / Debian 11+ / CentOS 8+ / 其他主流 Linux |
| **CPU**      | 1 核                                                    |
| **内存**     | 1 GB                                                    |
| **硬盘**     | 20 GB                                                   |
| **架构**     | x86_64 (AMD64)                                          |

### 推荐配置

| 项目     | 要求       |
| -------- | ---------- |
| **CPU**  | 2 核+      |
| **内存** | 2 GB+      |
| **硬盘** | 40 GB+ SSD |

### 需要开放的端口

| 端口 | 用途                           |
| ---- | ------------------------------ |
| 22   | SSH 远程连接                   |
| 80   | HTTP（如果用 Nginx 反向代理）  |
| 443  | HTTPS（如果用 Nginx 反向代理） |
| 9091 | 管理后台（如无反向代理）       |
| 9092 | 用户前端（如无反向代理）       |

> ⚠️ **注意**：后端 API（9090）和 Redis（6379）不对外暴露，仅在 Docker 内部网络通信，无需开放。

---

## 3. 安装 Docker （有安装docker的直接跳到第4步）

### 3.1 一键安装 Docker（推荐）

```bash
# 使用官方脚本一键安装 Docker
curl -fsSL https://get.docker.com | bash

# 国内服务器如果上面的命令太慢，可以使用阿里云镜像
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

### 3.2 启动 Docker 并设置开机自启

```bash
# 启动 Docker 服务
sudo systemctl start docker

# 设置开机自启
sudo systemctl enable docker
```

### 3.3 验证安装

```bash
# 检查 Docker 版本
docker --version
# 期望输出类似：Docker version 27.x.x

# 检查 Docker Compose 版本
docker compose version
# 期望输出类似：Docker Compose version v2.x.x
```

### 3.4 （可选）让当前用户免 sudo 使用 Docker

```bash
# 将当前用户添加到 docker 组
sudo usermod -aG docker $USER

# 重新登录生效（或执行以下命令）
newgrp docker
```

### 3.5 （国内服务器推荐）配置 Docker 镜像加速

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://docker.xuanyuan.me"
  ]
}
EOF

# 重启 Docker 使配置生效
sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

## 4. 准备部署文件

### 4.1 在服务器上创建部署目录

```bash
# 创建项目目录
sudo mkdir -p /opt/dujiao-next
cd /opt/dujiao-next
```

### 4.2 上传部署文件到服务器

请将本地准备好的 `deploy` 目录上传到服务器：

```bash
# 使用 scp 上传整个 deploy 目录到服务器
# 格式: scp -r <本地路径> <用户名>@<服务器IP>:<远程路径>
scp -r deploy/ root@你的服务器IP:/opt/dujiao-next/

# 如果你使用的是密钥登录
scp -r -i ~/.ssh/your_key deploy/ root@你的服务器IP:/opt/dujiao-next/
```

> 💡 **提示**：你也可以用 SFTP 工具（如 WinSCP、FileZilla）来可视化上传文件。

---

## 5. 修改配置文件

### 5.1 编辑配置文件

SSH 登录到服务器后：

```bash
cd /opt/dujiao-next
nano config.yml    # 或使用 vim config.yml
```

### 5.2 必须修改的配置项

以下是 **必须修改** 的关键配置，其他配置可保持默认：

```yaml
# ⚠️ 1. 运行模式：生产环境务必设为 release
server:
  mode: release

# ⚠️ 2. JWT 密钥：必须修改为随机字符串！
jwt:
  secret: 这里改成一串随机字符串比如abc123xyz456

user_jwt:
  secret: 这里改成另一串随机字符串比如def789uvw012
```

#### 如何生成随机密钥

```bash
# 在服务器终端执行，生成 32 位随机字符串
openssl rand -hex 16
# 输出示例: a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6

# 生成两个不同的密钥，分别填入 jwt.secret 和 user_jwt.secret
```

### 5.3 可选配置

```yaml
# 如果需要邮件功能（用户注册验证码等），启用并填写 SMTP 信息
email:
  enabled: true
  host: smtp.qq.com # QQ 邮箱
  port: 465
  username: 你的QQ邮箱@qq.com
  password: 你的邮箱授权码 # 注意：是授权码，不是邮箱密码
  from: 你的QQ邮箱@qq.com
  from_name: Dujiao-Next
  use_tls: false
  use_ssl: true
```

> ⚠️ **注意**：`redis.host` 和 `queue.host` 已经在配置文件中设置为 `redis`（Docker 容器名），**不要修改**这两个值！

---

## 6. 一键部署

```bash
# 登录服务器
ssh root@你的服务器IP

# 进入部署目录
cd /opt/dujiao-next

# 给部署脚本添加执行权限
chmod +x deploy.sh

# 执行一键部署
./deploy.sh
```

部署成功后你将看到类似输出：

```
╔══════════════════════════════════════════╗
║       Dujiao-Next Docker 部署脚本        ║
║            v0.0.1-beta                   ║
╚══════════════════════════════════════════╝

[1/6] 检查 Docker 环境...
✅ Docker 和 Docker Compose 已就绪
[2/6] 检查配置文件...
✅ 配置文件已就绪
[3/6] 检查项目文件...
✅ 项目文件已就绪
[4/6] 构建 Docker 镜像 (首次可能需要几分钟)...
✅ 镜像构建完成
[5/6] 启动所有服务...
✅ 服务已启动
[6/6] 等待服务就绪...

========== 服务状态 ==========
✅ 后端 API    : 运行中 (仅内部访问，端口 9090)
✅ 管理后台    : http://localhost:9091
✅ 用户前端    : http://localhost:9092

🎉 所有服务部署成功！
```

---

## 7. 手动部署（逐步操作）

如果你不想使用一键脚本，可以按以下步骤手动操作：

### 7.1 构建镜像

```bash
cd /opt/dujiao-next

# 构建所有镜像（首次需要下载依赖，大约 3-10 分钟）
docker compose build
```

### 7.2 启动服务

```bash
# 启动所有服务（-d 表示后台运行）
docker compose up -d
```

### 7.3 查看启动状态

```bash
# 查看所有容器状态
docker compose ps

# 期望输出：
# NAME            STATUS          PORTS
# dujiao-redis    Up (healthy)    6379/tcp
# dujiao-api      Up (healthy)    9090/tcp
# dujiao-admin    Up              0.0.0.0:9091->80/tcp
# dujiao-user     Up              0.0.0.0:9092->80/tcp
```

### 7.4 验证服务

```bash
# 测试后端 API（通过 docker exec，因为端口不对外暴露）
docker exec dujiao-api wget -qO- http://localhost:9090/health
# 期望输出: ok 或 {"status":"ok"} 等

# 测试管理后台
curl -I http://localhost:9091
# 期望返回: HTTP/1.1 200 OK

# 测试用户前端
curl -I http://localhost:9092
# 期望返回: HTTP/1.1 200 OK
```

---

## 8. 配置反向代理（域名 + HTTPS）

生产环境强烈建议使用 Nginx 反向代理 + HTTPS。以下提供两种方案：

### 方案 A：使用宿主机 Nginx + Let's Encrypt（推荐）

#### 8.1 安装 Nginx

```bash
# Ubuntu / Debian
sudo apt update && sudo apt install -y nginx

# CentOS
sudo yum install -y nginx
```

#### 8.2 安装 Certbot（自动获取免费 SSL 证书）

```bash
# Ubuntu / Debian
sudo apt install -y certbot python3-certbot-nginx

# CentOS
sudo yum install -y certbot python3-certbot-nginx
```

#### 8.3 配置 Nginx

假设你有以下域名：

- 用户前端：`shop.yourdomain.com`
- 管理后台：`admin.yourdomain.com`
- API：`api.yourdomain.com`（可选，也可以通过前端代理）

```bash
sudo nano /etc/nginx/sites-available/dujiao-next
```

写入以下内容：

```nginx
# ========== 用户前端 ==========
server {
    listen 80;
    server_name shop.yourdomain.com;   # ⬅️ 改成你的域名

    location / {
        proxy_pass http://127.0.0.1:9092;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # 上传文件大小限制
    client_max_body_size 10M;
}

# ========== 管理后台 ==========
server {
    listen 80;
    server_name admin.yourdomain.com;   # ⬅️ 改成你的域名

    location / {
        proxy_pass http://127.0.0.1:9091;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    client_max_body_size 10M;
}
```

```bash
# 启用配置
sudo ln -s /etc/nginx/sites-available/dujiao-next /etc/nginx/sites-enabled/

# 测试配置语法
sudo nginx -t

# 重载 Nginx
sudo systemctl reload nginx
```

#### 8.4 申请 SSL 证书（自动配置 HTTPS）

```bash
# 自动为你的域名申请 Let's Encrypt 免费证书
sudo certbot --nginx -d shop.yourdomain.com -d admin.yourdomain.com

# 按提示操作：
# 1. 输入你的邮箱
# 2. 同意条款
# 3. 选择是否将 HTTP 重定向到 HTTPS（建议选是）
```

#### 8.5 设置证书自动续期

```bash
# 测试自动续期
sudo certbot renew --dry-run

# Certbot 会自动添加定时任务，无需手动操作
```

### 方案 B：如果不想绑定域名（使用 IP 直接访问）

如果暂时不绑定域名，你可以直接通过 IP 访问：

```
用户前端：http://你的服务器IP:9092
管理后台：http://你的服务器IP:9091
```

> ⚠️ 后端 API（9090）不对外暴露，只需开放前端端口即可。

```bash
# 如果使用 firewalld
sudo firewall-cmd --permanent --add-port=9091/tcp
sudo firewall-cmd --permanent --add-port=9092/tcp
sudo firewall-cmd --reload

# 如果使用 ufw
sudo ufw allow 9091/tcp
sudo ufw allow 9092/tcp
```

---

## 9. 日常运维命令

> 以下所有命令都需要在 `/opt/dujiao-next` 目录下执行。

### 查看服务

```bash
# 查看所有容器运行状态
docker compose ps

# 查看所有服务的实时日志
docker compose logs -f

# 只查看后端 API 日志
docker compose logs -f api

# 只查看最近 100 行日志
docker compose logs --tail=100 api

# 查看某个容器的资源使用情况
docker stats dujiao-api dujiao-admin dujiao-user dujiao-redis
```

### 启停服务

```bash
# 停止所有服务（不删除容器）
docker compose stop

# 启动所有服务
docker compose start

# 重启所有服务
docker compose restart

# 只重启后端 API
docker compose restart api

# 停止并删除所有容器（数据卷保留，数据不丢失）
docker compose down

# ⚠️ 停止并删除所有容器 + 数据卷（会丢失数据！慎用！）
docker compose down -v
```

### 重新构建

```bash
# 代码更新后，重新构建并启动
docker compose build --no-cache
docker compose up -d

# 只重新构建某个服务
docker compose build --no-cache admin
docker compose up -d admin
```

### 进入容器调试

```bash
# 进入后端 API 容器
docker exec -it dujiao-api sh

# 进入 Redis 容器使用 redis-cli
docker exec -it dujiao-redis redis-cli
```

---

## 10. 数据备份与恢复

### 10.1 备份

```bash
#!/bin/bash
# 创建备份脚本: /opt/dujiao-next/backup.sh

BACKUP_DIR="/opt/dujiao-next/backups"
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p $BACKUP_DIR

echo "🔄 开始备份..."

# 1. 备份 SQLite 数据库
docker cp dujiao-api:/app/db/dujiao.db "$BACKUP_DIR/dujiao_${DATE}.db"

# 2. 备份上传文件
docker cp dujiao-api:/app/uploads "$BACKUP_DIR/uploads_${DATE}"

# 3. 备份配置文件
cp /opt/dujiao-next/config.yml "$BACKUP_DIR/config_${DATE}.yml"

# 4. 备份 Redis 数据
docker exec dujiao-redis redis-cli BGSAVE
sleep 2
docker cp dujiao-redis:/data/dump.rdb "$BACKUP_DIR/redis_${DATE}.rdb"

echo "✅ 备份完成: $BACKUP_DIR"
ls -lh $BACKUP_DIR/*${DATE}*
```

```bash
# 添加执行权限并运行
chmod +x /opt/dujiao-next/backup.sh
/opt/dujiao-next/backup.sh
```

#### 设置自动备份（每天凌晨 3 点）

```bash
# 编辑定时任务
crontab -e

# 添加以下行
0 3 * * * /opt/dujiao-next/backup.sh >> /opt/dujiao-next/backups/backup.log 2>&1
```

### 10.2 恢复

```bash
# 1. 停止后端服务
docker compose stop api

# 2. 恢复数据库
docker cp backups/dujiao_20260217_030000.db dujiao-api:/app/db/dujiao.db

# 3. 恢复上传文件
docker cp backups/uploads_20260217_030000/. dujiao-api:/app/uploads/

# 4. 重启服务
docker compose start api
```

---

## 11. 升级指南

当有新版本发布时，按以下步骤升级：

```bash
cd /opt/dujiao-next

# 1. 先备份！
./backup.sh

# 2. 停止当前服务
docker compose down

# 3. 更新文件
#    - 替换 backend/dujiao-next 为新版二进制文件
#    - 替换 admin/ 和 user/ 为新版前端源码
#    - 保留 config.yml 不要覆盖

# 4. 重新构建并启动
docker compose build --no-cache
docker compose up -d

# 5. 验证服务
docker compose ps
docker exec dujiao-api wget -qO- http://localhost:9090/health
```

---

## 12. 支付网关集成

### 12.1 BEpusdt 虚拟货币支付 (USDT)

Dujiao-Next 原生支持 **BEpusdt** 网关，可实现 USDT (TRC20/ERC20) 自动收款回调。

- **功能特性**：支持二维码支付、实时汇率转换、自动回调。
- **集成方式**：通过 Docker Compose 一键启动（需在 `docker-compose.yml` 中取消注释）。
- **详细指南**：请阅读独立文档 [BEpusdt 集成指南](BEPUSDT.md)

---

## 13. 常见问题排查

### ❓ Q1: 构建前端镜像时 `npm ci` 报错 / 太慢

**原因**：服务器网络访问 npm 仓库较慢。

**解决**：在 admin 和 user 的 Dockerfile 中加入 npm 镜像源：

```dockerfile
# 在 RUN npm ci 之前加一行
RUN npm config set registry https://registry.npmmirror.com
RUN npm ci
```

或者修改 Dockerfile：

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm config set registry https://registry.npmmirror.com && npm ci
COPY . .
RUN npm run build
```

---

### ❓ Q2: 后端启动失败 — `exec format error`

**原因**：二进制文件的架构与服务器不匹配。当前提供的是 `Linux x86_64` 版本。

**解决**：

```bash
# 检查服务器架构
uname -m
# 期望输出: x86_64

# 如果输出是 aarch64，说明你的服务器是 ARM 架构
# 需要使用 ARM 版本的二进制文件
```

---

### ❓ Q3: 前端页面空白 / API 请求 502

**原因**：前端 Nginx 无法连接到后端 API 容器。

**排查**：

```bash
# 1. 检查后端是否正常运行
docker compose ps
docker compose logs api

# 2. 从前端容器测试到后端的连接
docker exec dujiao-admin wget -qO- http://api:9090/health

# 3. 检查 Docker 网络
docker network inspect dujiao-next_dujiao-net
```

---

### ❓ Q4: Redis 连接失败

**排查**：

```bash
# 检查 Redis 是否运行
docker compose ps redis
docker compose logs redis

# 从 API 容器测试 Redis 连接
docker exec dujiao-api wget -qO- http://redis:6379 || echo "端口连通"

# 确认 config.yml 中 redis.host 和 queue.host 都是 "redis" 而非 "127.0.0.1"
```

---

### ❓ Q5: 容器启动后立即退出

```bash
# 查看退出日志
docker compose logs api

# 常见原因:
# 1. config.yml 格式错误 — 检查 YAML 缩进
# 2. 端口被占用
sudo lsof -i :8080
# 3. 权限问题
docker exec dujiao-api ls -la /app/
```

---

### ❓ Q6: 上传文件后图片不显示

**原因**：上传文件存储在容器内，需要通过 API 代理访问。

**排查**：

```bash
# 确认上传目录有数据
docker exec dujiao-api ls -la /app/uploads/

# 确认 nginx.conf 中 /uploads/ 的代理配置正确
# （deploy 目录中的 nginx.conf 已经配置好了）
```

---

### ❓ Q7: 如何查看 SQLite 数据库内容

```bash
# 方法 1: 将数据库拷贝到宿主机用工具查看
docker cp dujiao-api:/app/db/dujiao.db ./dujiao.db

# 方法 2: 进入容器安装 sqlite3
docker exec -it dujiao-api sh
apk add sqlite
sqlite3 /app/db/dujiao.db
# 执行 SQL: .tables   查看所有表
#           .schema   查看表结构
#           SELECT * FROM users;  查询数据
```

---

### ❓ Q8: 磁盘空间不足

```bash
# 清理未使用的 Docker 资源
docker system prune -a

# 查看 Docker 占用空间
docker system df
```

---

## 📝 部署清单（Checklist）

部署前请确认以下事项：

- [ ] 服务器已安装 Docker 和 Docker Compose
- [ ] 已上传所有部署文件到 `/opt/dujiao-next/`
- [ ] `backend/dujiao-next` 二进制文件已放置到位
- [ ] `admin/` 目录包含完整的管理后台源码
- [ ] `user/` 目录包含完整的用户前端源码
- [ ] 已修改 `config.yml` 中的 JWT 密钥
- [ ] `config.yml` 中 `server.mode` 已设置为 `release`
- [ ] `config.yml` 中 Redis 地址为 `redis`（不是 127.0.0.1）
- [ ] 服务器防火墙已开放所需端口
- [ ] （可选）已配置域名 DNS 解析
- [ ] （可选）已配置 HTTPS 证书

---

## 🆘 需要帮助？

如果在部署过程中遇到问题，请提供以下信息以便排查：

```bash
# 1. Docker 版本
docker --version
docker compose version

# 2. 服务器系统信息
uname -a
cat /etc/os-release

# 3. 容器运行状态
docker compose ps

# 4. 错误日志（最近 50 行）
docker compose logs --tail=50
```
