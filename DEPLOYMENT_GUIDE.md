# SWA 部署指南

## 📖 概述

本文档详细介绍了 SWA 应用在不同环境下的部署方法，包括开发环境、测试环境和生产环境的配置和部署步骤。

## 🏗️ 部署架构

### 系统要求

- **操作系统**: Linux (推荐 Ubuntu 20.04+), macOS, Windows
- **内存**: 最少 2GB，推荐 4GB+
- **存储**: 最少 10GB 可用空间
- **网络**: 支持 HTTP/HTTPS 访问

### 依赖服务

- **数据库**: MySQL 8.0+, PostgreSQL 12+, SQLite 3
- **缓存**: Redis 6.0+ (可选)
- **反向代理**: Nginx (推荐)

## 🚀 快速部署

### 1. 环境准备

```bash
# 安装 Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env

# 安装系统依赖 (Ubuntu/Debian)
sudo apt update
sudo apt install -y build-essential pkg-config libssl-dev

# 安装数据库 (MySQL)
sudo apt install -y mysql-server mysql-client

# 安装 Redis (可选)
sudo apt install -y redis-server
```

### 2. 应用部署

```bash
# 克隆项目
git clone <repository-url>
cd swa

# 构建应用
cargo build --release

# 构建插件
./build-plugin-simple.sh linux bas-site release

# 配置应用
cp config.yaml.example config.yaml
# 编辑配置文件
```

### 3. 启动应用

```bash
# 直接启动
./target/release/swa --config config.yaml

# 后台运行
nohup ./target/release/swa --config config.yaml > app.log 2>&1 &
```

## 🔧 配置管理

### 生产环境配置

```yaml
# config.yaml
# 数据库配置
database:
  host: localhost
  port: 3306
  username: swa_user
  password: "secure_password"
  database: swa_production
  pool_size: 20
  timeout: 30

# Redis配置
redis:
  host: localhost
  port: 6379
  password: "redis_password"
  db: 0
  pool_size: 10

# 应用配置
app:
  port: 8080
  host: "0.0.0.0"
  debug: false
  workers: 4

# 日志配置
logging:
  level: "info"
  file: "logs/app.log"
  max_size: "100MB"
  max_files: 10

# 安全配置
security:
  secret_key: "your-secret-key-here"
  session_timeout: 3600
  rate_limit: 1000

# IP访问控制
ip_access_control:
  enabled: false  # 生产环境通常关闭

# 缓存配置
cache:
  enabled: true
  ttl: 300
  max_size: "1GB"
```

### 环境变量配置

```bash
# 数据库配置
export SWA_DB_HOST=localhost
export SWA_DB_PORT=3306
export SWA_DB_USER=swa_user
export SWA_DB_PASSWORD=secure_password
export SWA_DB_NAME=swa_production

# Redis配置
export SWA_REDIS_HOST=localhost
export SWA_REDIS_PORT=6379
export SWA_REDIS_PASSWORD=redis_password

# 应用配置
export SWA_PORT=8080
export SWA_HOST=0.0.0.0
export SWA_DEBUG=false
export SWA_SECRET_KEY=your-secret-key-here
```

## 🐳 Docker 部署

### Dockerfile

```dockerfile
# 多阶段构建
FROM rust:1.70 as builder

WORKDIR /app
COPY . .

# 构建应用
RUN cargo build --release

# 构建插件
RUN ./build-plugin-simple.sh linux bas-site release

# 运行时镜像
FROM ubuntu:20.04

# 安装运行时依赖
RUN apt-get update && apt-get install -y \
    ca-certificates \
    libssl1.1 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# 复制构建产物
COPY --from=builder /app/target/release/swa /app/
COPY --from=builder /app/plugins/ /app/plugins/
COPY --from=builder /app/wwwroot/ /app/wwwroot/
COPY --from=builder /app/dist/ /app/dist/
COPY --from=builder /app/config.yaml /app/

# 创建日志目录
RUN mkdir -p /app/logs

# 暴露端口
EXPOSE 8080

# 启动应用
CMD ["./swa", "--config", "config.yaml"]
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SWA_DB_HOST=db
      - SWA_REDIS_HOST=redis
    depends_on:
      - db
      - redis
    volumes:
      - ./logs:/app/logs
      - ./uploads:/app/wwwroot/upload
    restart: unless-stopped

  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: swa_production
      MYSQL_USER: swa_user
      MYSQL_PASSWORD: secure_password
    volumes:
      - db_data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "3306:3306"
    restart: unless-stopped

  redis:
    image: redis:6.2-alpine
    command: redis-server --requirepass redis_password
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - app
    restart: unless-stopped

volumes:
  db_data:
  redis_data:
```

### 部署命令

```bash
# 构建和启动
docker-compose up -d

# 查看日志
docker-compose logs -f app

# 停止服务
docker-compose down

# 更新应用
docker-compose pull
docker-compose up -d
```

## 🌐 Nginx 配置

### 反向代理配置

```nginx
# /etc/nginx/sites-available/swa
upstream swa_backend {
    server 127.0.0.1:8080;
    # 如果有多个实例
    # server 127.0.0.1:8081;
    # server 127.0.0.1:8082;
}

server {
    listen 80;
    server_name your-domain.com;
    
    # 重定向到 HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;
    
    # SSL 配置
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
    
    # 静态文件处理
    location /static/ {
        alias /app/wwwroot/resources/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
    
    location /upload/ {
        alias /app/wwwroot/upload/;
        expires 1M;
        add_header Cache-Control "public";
    }
    
    # API 请求
    location /api/ {
        proxy_pass http://swa_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 超时配置
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }
    
    # 主应用
    location / {
        proxy_pass http://swa_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket 支持
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### 启用配置

```bash
# 创建软链接
sudo ln -s /etc/nginx/sites-available/swa /etc/nginx/sites-enabled/

# 测试配置
sudo nginx -t

# 重载配置
sudo systemctl reload nginx
```

## 🔄 持续集成/持续部署 (CI/CD)

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy SWA

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions-rs/toolchain@v1
        with:
          toolchain: stable
      
      - name: Run tests
        run: cargo test
      
      - name: Build application
        run: cargo build --release
      
      - name: Build plugins
        run: |
          chmod +x build-plugin-simple.sh
          ./build-plugin-simple.sh linux bas-site release

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to server
        uses: appleboy/ssh-action@v0.1.5
        with:
          host: ${{ secrets.HOST }}
          username: ${{ secrets.USERNAME }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /app/swa
            git pull origin main
            cargo build --release
            ./build-plugin-simple.sh linux bas-site release
            sudo systemctl restart swa
```

### 部署脚本

```bash
#!/bin/bash
# deploy.sh

set -e

echo "🚀 开始部署 SWA 应用..."

# 备份当前版本
if [ -d "/app/swa" ]; then
    echo "📦 备份当前版本..."
    sudo cp -r /app/swa /app/swa.backup.$(date +%Y%m%d_%H%M%S)
fi

# 拉取最新代码
echo "📥 拉取最新代码..."
cd /app/swa
git pull origin main

# 构建应用
echo "🔨 构建应用..."
cargo build --release

# 构建插件
echo "🔌 构建插件..."
./build-plugin-simple.sh linux bas-site release

# 运行数据库迁移
echo "🗄️ 运行数据库迁移..."
# ./migrate.sh

# 重启服务
echo "🔄 重启服务..."
sudo systemctl restart swa

# 检查服务状态
echo "✅ 检查服务状态..."
sleep 5
sudo systemctl status swa

echo "🎉 部署完成!"
```

## 📊 监控和日志

### 系统服务配置

```ini
# /etc/systemd/system/swa.service
[Unit]
Description=SWA Web Application
After=network.target mysql.service redis.service

[Service]
Type=simple
User=swa
Group=swa
WorkingDirectory=/app/swa
ExecStart=/app/swa/target/release/swa --config config.yaml
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal
SyslogIdentifier=swa

# 环境变量
Environment=RUST_LOG=info
Environment=SWA_ENV=production

[Install]
WantedBy=multi-user.target
```

### 日志管理

```bash
# 查看应用日志
sudo journalctl -u swa -f

# 查看应用日志文件
tail -f /app/swa/logs/$(date +%Y%m%d)/output.log

# 日志轮转配置
sudo vim /etc/logrotate.d/swa
```

### 监控脚本

```bash
#!/bin/bash
# monitor.sh

# 检查应用状态
check_app() {
    if curl -f http://localhost:8080/health > /dev/null 2>&1; then
        echo "✅ 应用运行正常"
        return 0
    else
        echo "❌ 应用无响应"
        return 1
    fi
}

# 检查数据库连接
check_database() {
    if mysql -h localhost -u swa_user -p$SWA_DB_PASSWORD -e "SELECT 1" > /dev/null 2>&1; then
        echo "✅ 数据库连接正常"
        return 0
    else
        echo "❌ 数据库连接失败"
        return 1
    fi
}

# 检查 Redis 连接
check_redis() {
    if redis-cli -h localhost -a $SWA_REDIS_PASSWORD ping > /dev/null 2>&1; then
        echo "✅ Redis 连接正常"
        return 0
    else
        echo "❌ Redis 连接失败"
        return 1
    fi
}

# 主检查逻辑
main() {
    echo "🔍 检查系统状态..."
    
    check_app || exit 1
    check_database || exit 1
    check_redis || exit 1
    
    echo "🎉 所有服务运行正常"
}

main
```

## 🔒 安全配置

### 防火墙配置

```bash
# UFW 配置
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw enable
```

### SSL 证书配置

```bash
# 使用 Let's Encrypt
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d your-domain.com
```

### 安全加固

```bash
# 创建专用用户
sudo useradd -r -s /bin/false swa
sudo chown -R swa:swa /app/swa

# 设置文件权限
sudo chmod 755 /app/swa/target/release/swa
sudo chmod 644 /app/swa/config.yaml
```

## 🚨 故障排除

### 常见问题

1. **应用启动失败**
   ```bash
   # 检查配置文件
   ./target/release/swa --config config.yaml --check-config
   
   # 检查端口占用
   sudo netstat -tlnp | grep :8080
   ```

2. **数据库连接失败**
   ```bash
   # 检查数据库服务
   sudo systemctl status mysql
   
   # 测试连接
   mysql -h localhost -u swa_user -p
   ```

3. **插件加载失败**
   ```bash
   # 检查插件文件
   ls -la plugins/libswa_plugin_*.so
   
   # 检查依赖
   ldd plugins/libswa_plugin_*.so
   ```

### 性能优化

```bash
# 调整系统参数
echo 'net.core.somaxconn = 65535' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_max_syn_backlog = 65535' >> /etc/sysctl.conf
sysctl -p

# 调整应用参数
export RUST_LOG=warn  # 减少日志输出
export SWA_WORKERS=4  # 调整工作线程数
```

## 📚 参考资源

- [Rust 部署最佳实践](https://doc.rust-lang.org/book/ch14-00-more-about-cargo.html)
- [Actix-web 部署指南](https://actix.rs/docs/server/)
- [Nginx 配置参考](https://nginx.org/en/docs/)
- [Docker 最佳实践](https://docs.docker.com/develop/dev-best-practices/)

---

**Happy Deployment!** 🚀
