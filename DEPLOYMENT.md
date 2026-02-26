# Strapi CMS 阿里云服务器部署文档

## 1. 项目概述

本项目是基于 Strapi 5.35.0 版本的内容管理系统，使用 TypeScript 开发，默认使用 SQLite 数据库。项目包含多个 API 端点，用于管理网站的各类内容，如文章、服务、关于我们等。

## 2. 服务器配置要求

根据 Strapi 官方文档和项目需求，推荐以下服务器配置：

| 环境 | CPU | 内存 | 存储 | 带宽 | 操作系统 |
|------|-----|------|------|------|----------|
| 开发/测试 | 1核 | 2GB | 40GB SSD | 1M | Ubuntu 20.04 LTS |
| 生产环境 | 2核 | 4GB+ | 80GB SSD | 5M+ | Ubuntu 20.04 LTS |

## 3. 环境准备

### 3.1 购买阿里云服务器

1. 登录阿里云官网，进入控制台
2. 选择「云服务器 ECS」
3. 点击「创建实例」
4. 选择适合的配置，推荐使用 Ubuntu 20.04 LTS 操作系统
5. 完成支付，获取服务器公网 IP

### 3.2 配置安全组

1. 进入 ECS 实例详情页
2. 点击「安全组」→「配置规则」
3. 添加以下规则：
   - 端口 22 (SSH)：允许所有 IP 访问
   - 端口 80 (HTTP)：允许所有 IP 访问
   - 端口 443 (HTTPS)：允许所有 IP 访问
   - 端口 1337 (Strapi)：允许所有 IP 访问

### 3.3 连接服务器

使用 SSH 工具连接服务器：

```bash
ssh root@服务器公网IP
```

## 4. 环境安装

### 4.1 更新系统

```bash
apt update && apt upgrade -y
```

### 4.2 安装 Node.js

Strapi 5.35.0 要求 Node.js 20.0.0 或以上版本：

```bash
# 安装 Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs

# 验证安装
node -v
npm -v

# 安装 pnpm（项目使用 pnpm 作为包管理器）
npm install -g pnpm
```

### 4.3 安装构建工具

```bash
apt install -y build-essential python3
```

## 5. 项目部署

### 5.1 克隆项目

```bash
# 创建项目目录
mkdir -p /var/www/strapi
cd /var/www/strapi

# 克隆项目（如果使用 Git）
git clone <项目仓库地址> .

# 或上传项目文件
# 使用 scp 或 WinSCP 等工具将本地项目文件上传到服务器
```

### 5.2 安装依赖

```bash
cd /var/www/strapi
pnpm install
```

### 5.3 配置环境变量

复制 `.env.example` 文件并修改：

```bash
cp .env.example .env

# 编辑 .env 文件
nano .env
```

修改以下配置：

```env
HOST=0.0.0.0
PORT=1337
APP_KEYS="your-secret-key-1,your-secret-key-2"
API_TOKEN_SALT=your-api-token-salt
ADMIN_JWT_SECRET=your-admin-jwt-secret
TRANSFER_TOKEN_SALT=your-transfer-token-salt
JWT_SECRET=your-jwt-secret
ENCRYPTION_KEY=your-encryption-key

# 数据库配置（如果使用 MySQL 或 PostgreSQL）
# DATABASE_CLIENT=mysql
# DATABASE_HOST=localhost
# DATABASE_PORT=3306
# DATABASE_NAME=strapi
# DATABASE_USERNAME=strapi
# DATABASE_PASSWORD=your-database-password
```

### 5.4 构建项目

```bash
NODE_ENV=production pnpm build
```

### 5.5 启动服务

#### 5.5.1 使用 PM2 管理进程

```bash
# 安装 PM2
npm install -g pm2

# 启动服务
pm run start

# 或使用 PM2 启动
pm run start:pm2

# 查看进程状态
pm run pm2:logs
```

#### 5.5.2 配置系统服务（可选）

创建 systemd 服务文件：

```bash
nano /etc/systemd/system/strapi.service
```

添加以下内容：

```ini
[Unit]
Description=Strapi CMS
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/var/www/strapi
ExecStart=/usr/bin/npm start
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

启用并启动服务：

```bash
systemctl daemon-reload
systemctl enable strapi
systemctl start strapi
systemctl status strapi
```

## 6. 数据库配置

### 6.1 SQLite 数据库（默认）

项目默认使用 SQLite 数据库，无需额外配置。数据库文件位于 `.tmp/data.db`。

### 6.2 切换到 MySQL 数据库（推荐生产环境）

1. 安装 MySQL：

```bash
apt install -y mysql-server
mysql_secure_installation
```

2. 创建数据库和用户：

```bash
mysql -u root -p

CREATE DATABASE strapi CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'strapi'@'localhost' IDENTIFIED BY 'your-password';
GRANT ALL PRIVILEGES ON strapi.* TO 'strapi'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

3. 修改 `.env` 文件：

```env
DATABASE_CLIENT=mysql
DATABASE_HOST=localhost
DATABASE_PORT=3306
DATABASE_NAME=strapi
DATABASE_USERNAME=strapi
DATABASE_PASSWORD=your-password
```

4. 重新构建项目：

```bash
NODE_ENV=production pnpm build
pm run start
```

### 6.3 切换到 PostgreSQL 数据库（推荐）

1. 安装 PostgreSQL：

```bash
apt install -y postgresql postgresql-contrib
```

2. 创建数据库和用户：

```bash
sudo -u postgres psql

CREATE DATABASE strapi;
CREATE USER strapi WITH PASSWORD 'your-password';
ALTER ROLE strapi SUPERUSER;

```

3. 修改 `.env` 文件：

```env
DATABASE_CLIENT=postgres
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=strapi
DATABASE_USERNAME=strapi
DATABASE_PASSWORD=your-password
```

4. 重新构建项目：

```bash
NODE_ENV=production pnpm build
npm run start
```

## 7. 安全设置

### 7.1 Nginx 反向代理

安装并配置 Nginx：

```bash
apt install -y nginx

# 创建 Nginx 配置文件
nano /etc/nginx/sites-available/strapi
```

添加以下内容：

```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:1337;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

启用配置并重启 Nginx：

```bash
ln -s /etc/nginx/sites-available/strapi /etc/nginx/sites-enabled/
nginx -t
systemctl restart nginx
```

### 7.2 SSL 证书配置

使用 Let's Encrypt 免费 SSL 证书：

```bash
apt install -y certbot python3-certbot-nginx
certbot --nginx -d your-domain.com
```

### 7.3 防火墙设置

```bash
ufw allow 22
ufw allow 80
ufw allow 443
ufw enable
```

## 8. 性能优化

### 8.1 服务器优化

1. 增加 Node.js 内存限制：

```bash
export NODE_OPTIONS="--max-old-space-size=2048"
```

2. 配置 PM2 集群模式：

```bash
pm run start:pm2-cluster
```

### 8.2 数据库优化

- 使用 PostgreSQL 数据库（生产环境推荐）
- 配置数据库连接池
- 定期备份数据库

### 8.3 缓存设置

配置 Nginx 缓存：

```nginx
location / {
    # 其他配置...
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=strapi_cache:10m max_size=100m inactive=60m;
    proxy_cache strapi_cache;
    proxy_cache_valid 200 302 60m;
    proxy_cache_valid 404 1m;
}
```

## 9. 常见问题及解决方案

### 9.1 内存不足

**症状**：部署失败，出现 OOM 错误
**解决方案**：
- 增加服务器内存
- 优化 Node.js 内存使用
- 使用 PM2 集群模式

### 9.2 数据库连接失败

**症状**：无法连接数据库
**解决方案**：
- 检查数据库服务是否运行
- 验证数据库连接参数
- 检查防火墙设置

### 9.3 权限问题

**症状**：文件或目录权限错误
**解决方案**：
- 检查文件权限
- 使用正确的用户运行服务

### 9.4 端口占用

**症状**：端口已被占用
**解决方案**：
- 检查端口使用情况：`lsof -i :1337`
- 停止占用端口的进程
- 或修改 Strapi 端口配置

## 10. 维护与监控

### 10.1 日志管理

- Strapi 日志：`/var/www/strapi/logs`
- Nginx 日志：`/var/log/nginx`

### 10.2 定期备份

1. 数据库备份：

```bash
# MySQL
mysqldump -u strapi -p strapi > strapi_backup.sql

# PostgreSQL
pg_dump -U strapi strapi > strapi_backup.sql

# SQLite
cp /var/www/strapi/.tmp/data.db /path/to/backup/
```

2. 项目文件备份：

```bash
tar -czvf strapi_backup.tar.gz /var/www/strapi
```

### 10.3 监控

- 使用 PM2 监控进程状态：`pm2 status`
- 使用阿里云控制台监控服务器状态
- 配置监控告警

## 11. 升级与更新

### 11.1 项目更新

1. 停止服务：

```bash
npm run stop
```

2. 拉取最新代码：

```bash
git pull
```

3. 安装依赖：

```bash
pnpm install
```

4. 构建项目：

```bash
NODE_ENV=production pnpm build
```

5. 启动服务：

```bash
npm run start
```

### 11.2 Strapi 版本升级

```bash
# 升级 Strapi
pnpm upgrade @strapi/strapi @strapi/plugin-users-permissions @strapi/plugin-cloud

# 构建项目
NODE_ENV=production pnpm build

# 启动服务
npm run start
```

## 12. 部署检查清单

- [ ] 服务器配置满足要求
- [ ] 安全组规则已配置
- [ ] Node.js 环境已安装
- [ ] 项目依赖已安装
- [ ] 环境变量已配置
- [ ] 项目已构建
- [ ] 服务已启动
- [ ] Nginx 反向代理已配置
- [ ] SSL 证书已安装
- [ ] 防火墙已配置
- [ ] 数据库已配置
- [ ] 备份策略已设置
- [ ] 监控已配置

## 13. 附录

### 13.1 常用命令

```bash
# 启动服务
npm run start

# 停止服务
npm run stop

# 查看日志
npm run logs

# 重启服务
npm run restart

# 构建项目
npm run build
```

### 13.2 项目结构

```
strapi-cms/
├── config/          # 配置文件
├── data/            # 数据文件
├── database/        # 数据库迁移文件
├── public/          # 静态文件
├── scripts/         # 脚本文件
├── src/             # 源代码
│   ├── api/         # API 定义
│   ├── components/  # 组件
│   └── admin/       # 管理后台
├── types/           # TypeScript 类型定义
├── .env             # 环境变量
├── package.json     # 项目配置
└── pnpm-lock.yaml   # 依赖锁定文件
```

### 13.3 环境变量说明

| 变量名 | 说明 | 默认值 |
|-------|------|-------|
| HOST | 服务器主机 | 0.0.0.0 |
| PORT | 服务器端口 | 1337 |
| APP_KEYS | 应用密钥 | 需修改 |
| API_TOKEN_SALT | API 令牌盐 | 需修改 |
| ADMIN_JWT_SECRET | 管理后台 JWT 密钥 | 需修改 |
| TRANSFER_TOKEN_SALT | 传输令牌盐 | 需修改 |
| JWT_SECRET | JWT 密钥 | 需修改 |
| ENCRYPTION_KEY | 加密密钥 | 需修改 |
| DATABASE_CLIENT | 数据库客户端 | sqlite |
| DATABASE_HOST | 数据库主机 | localhost |
| DATABASE_PORT | 数据库端口 | 3306 (MySQL) / 5432 (PostgreSQL) |
| DATABASE_NAME | 数据库名称 | strapi |
| DATABASE_USERNAME | 数据库用户名 | strapi |
| DATABASE_PASSWORD | 数据库密码 | 需设置 |

---

**注意**：本部署文档基于 Strapi 5.35.0 版本，具体步骤可能因版本更新而有所变化。请参考官方文档获取最新部署指南。