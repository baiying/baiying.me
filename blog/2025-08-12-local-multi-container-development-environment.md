---
slug: guide-to-deploying-local-multi-container-development-environment
title: 本地多容器开发环境部署与 HTTPS 配置指南
authors: baiying
tags: [docker, local, development]
---

## 1. 项目结构与 `docker-compose.yml`

本地项目包含 4 个服务：

* **api**：Python FastAPI 应用（使用 `uvicorn` 运行）
* **console**：前端管理控制台（Node.js + pnpm）
* **www**：前端网站（Next.js + pnpm）
* **nginx**：统一反向代理和域名路由

示例 `docker-compose.yml`：

```yaml
services:
  api:
    build:
      context: ./api.aigc.pub
      dockerfile: Dockerfile.dev
    container_name: api
    volumes:
      - ./api.aigc.pub:/app
    ports:
      - "7080:7080"
    networks:
      - aigc_net
    command: uv run uvicorn main:app --host 0.0.0.0 --port 7080 --reload

  console:
    build:
      context: ./console.aigc.pub
      dockerfile: Dockerfile.dev
    container_name: console
    ports:
      - "3200:3200"
    networks:
      - aigc_net
    command: pnpm dev

  www:
    build:
      context: ./www.aigc.pub
      dockerfile: Dockerfile.dev
    container_name: www
    ports:
      - "3100:3100"
    networks:
      - aigc_net
    command: pnpm dev

  nginx:
    build:
      context: ./nginx
    container_name: nginx
    ports:
      - "80:80"
      - "443:443" # 或者 "8443:443" 避免与 macOS 占用端口冲突
    depends_on:
      - api
      - console
      - www
    networks:
      - aigc_net

networks:
  aigc_net:
    driver: bridge
```

> 注意：`ports` 需写成 `"宿主机端口:容器端口"` 的字符串格式，否则会报错 `invalid containerPort`。

---

## 2. `.dockerignore` 推荐写法

在根目录添加 `.dockerignore` 避免无关文件进入构建上下文：

```dockerignore
# Node.js 依赖
node_modules
npm-debug.log
pnpm-lock.yaml
yarn.lock

# Python 虚拟环境
.venv
__pycache__/
*.pyc

# 系统文件
.DS_Store
.env
*.log

# 构建产物
dist
build
.cache
```

---

## 3. 本地域名绑定

在 macOS `/etc/hosts` 添加：

```
127.0.0.1 www.aigc.pub
127.0.0.1 console.aigc.pub
127.0.0.1 api.aigc.pub
```

---

## 4. Nginx 域名路由配置

在 `nginx.conf` 中添加反向代理：

```nginx
server {
    listen 80;
    server_name console.aigc.pub;
    location / {
        proxy_pass http://console:3200;
    }
}

server {
    listen 80;
    server_name www.aigc.pub;
    location / {
        proxy_pass http://www:3100;
    }
}

server {
    listen 80;
    server_name api.aigc.pub;
    location / {
        proxy_pass http://api:7080;
    }
}
```

---

## 5. 本地 HTTPS 配置

### 方法 1：使用 mkcert 生成本地可信证书

1. 安装 mkcert

```bash
brew install mkcert nss
mkcert -install
```

2. 生成证书（在 `nginx/certs/` 目录下）

```bash
mkcert console.aigc.pub www.aigc.pub api.aigc.pub
```

3. 修改 Nginx 配置支持 HTTPS

```nginx
server {
    listen 443 ssl;
    server_name console.aigc.pub;

    ssl_certificate /etc/nginx/certs/console.aigc.pub.pem;
    ssl_certificate_key /etc/nginx/certs/console.aigc.pub-key.pem;

    location / {
        proxy_pass http://console:3200;
    }
}
```

4. 在 `docker-compose.yml` 中暴露 443 端口：

```yaml
ports:
  - "80:80"
  - "443:443" # 或 "8443:443"
```

---

### 方法 2：使用自签名证书

```bash
mkdir -p nginx/certs
openssl req -x509 -nodes -days 365 \
  -newkey rsa:2048 \
  -keyout nginx/certs/selfsigned.key \
  -out nginx/certs/selfsigned.crt
```

Nginx 中引用：

```nginx
ssl_certificate /etc/nginx/certs/selfsigned.crt;
ssl_certificate_key /etc/nginx/certs/selfsigned.key;
```

---

## 6. 常见问题排查

### 502 Bad Gateway

* 检查目标容器是否正常运行：`docker ps`
* 确认服务在容器内可访问（在 nginx 容器中执行）：

```bash
curl http://console:3200
curl http://www:3100
curl http://api:7080
```

* 确认 `nginx.conf` 的 `proxy_pass` 指向服务容器名+端口

### `invalid containerPort: 443`

* 确保写法是 `"443:443"`（字符串格式）
* 避免只写 `- 443`
* 在 macOS 上如被系统占用，可改为 `"8443:443"`

---

## 7. 启动项目

```bash
docker compose up --build
```

