---
title: "Ubuntu 22.04 下的 Ghost 安装笔记"
date: "2023-08-06"
description: ""
tags: "Ubuntu, Ghost"
---

系统 Ubuntu 22.04

## 安装 `certbot` 

```bash
sudo apt install software-properties-common
sudo add-apt-repository ppa:certbot/certbot
sudo apt update
sudo apt install certbot
sudo certbot certonly --standalone -d YOUR_DOMAIN
```

## 安装 Docker 和 Docker Compose

使用以下命令下载 `docker-compose` 安装文件 (记得将版本号更换为[官方发布](https://github.com/docker/compose/releases)的最新版)

```bash
mkdir -p ~/.docker/cli-plugins/
curl -SL https://github.com/docker/compose/releases/download/v2.12.2/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose
```

给下载的安装文件授予权限：

```bash
chmod +x ~/.docker/cli-plugins/docker-compose
```

验证是否安装成功：

```bash
docker compose version
```

## 安装 Ghost

```bash
mkdir ghost && cd ghost
```

创建 `docker-compose.yml` 并在文本编辑器中打开，复制粘贴如下内容：

```yaml
version: '3'
services:

  ghost:
    image: ghost:latest
    restart: always
    depends_on:
      - db
    environment:
      url: https://example.com
      database__client: mysql
      database__connection__host: db
      database__connection__user: root
      database__connection__password: YOUR_DATABASE_ROOT_PASSWORD
      database__connection__database: ghost
    volumes:
      - /opt/ghost_content:/var/lib/ghost/content

  db:
    image: mysql:8
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: YOUR_DATABASE_ROOT_PASSWORD
    volumes:
      - /opt/ghost_mysql:/var/lib/mysql

  nginx:
    build:
      context: ./nginx
      dockerfile: Dockerfile
    restart: always
    depends_on:
      - ghost
    ports:
      - "80:80"
      - "443:443"
    volumes:
       - /etc/letsencrypt/:/etc/letsencrypt/
       - /usr/share/nginx/html:/usr/share/nginx/html
```


## 创建 Nginx Docker 镜像

Docker Compose 文件依赖于自定义的 Nginx 镜像。此镜像将与适当的服务器块设置打包在一起。

1. 在 `ghost` 文件夹下创建 `nginx` 文件夹

```bash
mkdir nginx
```

2. 创建一个名为 `Dockerfile` 的文件并复制粘贴如下内容：

```dockerfile
FROM nginx:latest
COPY default.conf /etc/nginx/conf.d
```

3. 创建一个名为 `default.conf` 的文件并复制粘贴如下内容：

```dockerfile
server {
  listen 80;
  listen [::]:80;
  server_name YOUR_DOMAIN;
  # Useful for Let's Encrypt
  location /.well-known/acme-challenge/ { root /usr/share/nginx/html; allow all; }
  location / { return 301 https://$host$request_uri; }
}

server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  server_name YOUR_DOMAIN;

  ssl_protocols TLSv1.2;
  ssl_ciphers HIGH:!MEDIUM:!LOW:!aNULL:!NULL:!SHA;
  ssl_prefer_server_ciphers on;
  ssl_session_cache shared:SSL:10m;

  ssl_certificate     /etc/letsencrypt/live/YOUR_DOMAIN/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/YOUR_DOMAIN/privkey.pem;

  location / {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-Proto https;
    proxy_pass http://ghost:2368;
  }
}
```

## 运行并测试

```bash
docker compose up -d
```

## 更新镜像

```bash
docker compose down && docker compose pull && docker compose up -d
```

## 设置定期自签 SSL 证书

在文本编辑器中打开 `crontab`

```bash
sudo crontab -e
```

在打开的文件中添加如下一行：

```
0 23 * * * certbot certonly -n --webroot -w /usr/share/nginx/html -d YOUR_DOMAIN --deploy-hook='docker exec ghost_nginx_1 nginx -s reload'
```

你也可以通过 `--dry-run` 选项测试新添加的定时命令

```bash
sudo bash -c "certbot certonly -n --webroot -w /usr/share/nginx/html -d example.com --deploy-hook='docker exec ghost_nginx_1 nginx -s reload'"
```