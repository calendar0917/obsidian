---
title: "御林服务配置合集"
subtitle: "御林服务配置合集"
summary: "整合了各种服务的配置文件，方便复查"
description: "整合了各种服务的配置文件，方便复查"
image: ""
date: 2025-11-23
lastmod: 2025-11-23
draft: false
toc:
 enable: true
hiddenFromHomePage: false
weight: false
categories: ["DevOps"]
tags: ["技术杂项"]
---

## /etc/nginx

3000 homepage

3030 outline

9000 authentik

9111 portainer

8484 grist

51821 wg-gen-web

### nginx.conf

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 768;
        # multi_accept on;
}

http {

        ##
        # Basic Settings
        ##

        sendfile on;
        tcp_nopush on;
        types_hash_max_size 2048;
        # server_tokens off;

        # server_names_hash_bucket_size 64;
        # server_name_in_redirect off;

        include /etc/nginx/mime.types;
        default_type application/octet-stream;

        ##
        # SSL Settings
        ##

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3; # Dropping SSLv3, ref: POODLE
        ssl_prefer_server_ciphers on;

        ##
        # Logging Settings
        ##

        access_log /var/log/nginx/access.log;

        ##
        # Gzip Settings
        ##

        gzip on;

        # gzip_vary on;
        # gzip_proxied any;
        # gzip_comp_level 6;
        # gzip_buffers 16 8k;
        # gzip_http_version 1.1;
        # gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

        ##
        # Virtual Host Configs
        ##

        include /etc/nginx/conf.d/*.conf;
        include /etc/nginx/sites-enabled/*;
}


#mail {
#       # See sample authentication script at:
#       # http://wiki.nginx.org/ImapAuthenticateWithApachePhpScript
#
#       # auth_http localhost/auth.php;
#       # pop3_capabilities "TOP" "USER";
#       # imap_capabilities "IMAP4rev1" "UIDPLUS";
#
#       server {
#               listen     localhost:110;
#               protocol   pop3;
#               proxy      on;
#       }
#
#       server {
#               listen     localhost:143;
#               protocol   imap;
#               proxy      on;
#       }
#}
```

### conf.d/auth.conf

```nginx
 # Upstream where your authentik server is hosted.
upstream authentik {
    server 127.0.0.1:9443;
    # Improve performance by keeping some connections alive.
    keepalive 10;
}

# Upgrade WebSocket if requested, otherwise use keepalive
map $http_upgrade $connection_upgrade_keepalive {
    default upgrade;
    ''      '';
}

server {
    # HTTP server config
    server_name auth.yulinsec.cn;

    #Real IP From Frp
    set_real_ip_from 127.0.0.0/8;
    real_ip_header proxy_protocol;

    # Proxy site
    # Location can be set to a subpath if desired, see documentation linked below:
    # https://docs.goauthentik.io/docs/install-config/configuration/#authentik_web__path
    location / {
        proxy_pass http://127.0.0.1:9000;
        proxy_http_version 1.1;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $http_host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade_keepalive;
    }

    listen 443 ssl proxy_protocol; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/auth.yulinsec.cn/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/auth.yulinsec.cn/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}


server {
    if ($host = auth.yulinsec.cn) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    listen 80 proxy_protocol;
    server_name auth.yulinsec.cn;
    return 404; # managed by Certbot
}
```

### conf.d/outline.conf

```nginx
server {
  server_name docs.yulinsec.cn;

  set_real_ip_from 127.0.0.0/8;
  real_ip_header proxy_protocol;

  location / {
    proxy_pass http://127.0.0.1:3030/;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_set_header Host $host;

    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Scheme $scheme;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_redirect off;
  }

    listen 443 ssl proxy_protocol; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/docs.yulinsec.cn/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/docs.yulinsec.cn/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = docs.yulinsec.cn) {
        return 301 https://$host$request_uri;
    } # managed by Certbot
  server_name docs.yulinsec.cn;
  listen 80 proxy_protocol;
    return 404; # managed by Certbot
}
```

### /conf.d/portainer.conf

```NG
server {   # 启用 proxy protocol
    server_name portainer.yulinsec.cn;

    # 处理 proxy protocol 后可继续使用真实客户端 IP
    real_ip_header proxy_protocol;
    set_real_ip_from 127.0.0.0/8;   # 按需限制允许发送 proxy protocol 的来源

    location / {
        proxy_pass http://127.0.0.1:9111;

        # 传递代理头部（客户端真实信息）
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $proxy_protocol_addr;
        proxy_set_header X-Forwarded-For $proxy_protocol_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    listen 443 ssl proxy_protocol; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/portainer.yulinsec.cn/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/portainer.yulinsec.cn/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = portainer.yulinsec.cn) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    listen 80 proxy_protocol;
    server_name portainer.yulinsec.cn;
    return 404; # managed by Certbot


}
```

### conf.d/grist.conf

```nginx
server {
    # 配置域名
    server_name grist.yulinsec.cn;

    # 处理真实 IP（保持与其他服务一致）
    set_real_ip_from 127.0.0.0/8;
    real_ip_header proxy_protocol;

    # 反向代理到 grist 服务端口
    location / {
        proxy_pass http://127.0.0.1:8484;  # 替换为实际端口

        # 通用代理头部配置（保持与其他服务一致）
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
        # WebSocket支持（Grist实时协作依赖）
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400;
    }

    # HTTPS 配置（通过 Certbot 自动生成）
    listen 443 ssl proxy_protocol;
    ssl_certificate /etc/letsencrypt/live/grist.yulinsec.cn/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/grist.yulinsec.cn/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

# HTTP 自动跳转 HTTPS
server {
    if ($host = grist.yulinsec.cn) {
        return 301 https://$host$request_uri;
    }

    listen 80 proxy_protocol;
    server_name grist.yulinsec.cn;
    return 404;
}
```

### conf.d/homepage.conf

```nginx
# /etc/nginx/conf.d/homepage.conf
server {
    server_name homepage.yulinsec.cn;  # 自定义域名

    # 处理真实IP（与其他服务保持一致）
    set_real_ip_from 127.0.0.0/8;
    real_ip_header proxy_protocol;

    location / {
        proxy_pass http://127.0.0.1:3000;  # 代理到本地3000端口

        # 通用代理头部配置（与其他服务保持一致）
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
    }

    # HTTPS配置（通过Certbot生成）
    #listen 443 ssl proxy_protocol;
    #ssl_certificate /etc/letsencrypt/live/homepage.yulinsec.cn/fullchain.pem;
    #ssl_certificate_key /etc/letsencrypt/live/homepage.yulinsec.cn/privkey.pem;
    #include /etc/letsencrypt/options-ssl-nginx.conf;
    #ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

# HTTP自动跳转HTTPS
server {
    if ($host = homepage.yulinsec.cn) {
        return 301 https://$host$request_uri;
    }

    listen 80 proxy_protocol;
    server_name homepage.yulinsec.cn;
    return 404;
}
```

```bash
# 生成证书
sudo certbot --nginx -d homepage.yulinsec.cn

# 检查配置是否有误
sudo nginx -t

# 重启 Nginx 生效
sudo systemctl restart nginx
```

### conf.d/wgeasy.conf

```nginx
server {
    server_name wg-easy.yulinsec.cn; # ⚠️ 修改为你的域名
    
    location / {
    proxy_pass http://127.0.0.1:8080/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_set_header Host $host;
}


    listen 443 ssl proxy_protocol; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/wg-easy.yulinsec.cn/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/wg-easy.yulinsec.cn/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = wg-easy.yulinsec.cn) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


server_name wg-easy.yulinsec.cn;
    listen 80 proxy_protocol;
    return 404; # managed by Certbot


}
```

## /root/frp

### frpc.toml

```toml
# 服务器公网地址
serverAddr = "47.242.81.180"
serverPort = 15443
transport.protocol = "tcp"
loginFailExit = false

# 鉴权配置
[auth]
method = "token"
token = "……"

# 日志配置
[log]
level = "info"
to = "/root/frp/logs/frpc.log"

# 代理配置
[[proxies]]
name = "auth-yulinsec-cn-http"
type = "http"
localIP = "127.0.0.1"
localPort = 80
customDomains = ["auth.yulinsec.cn"]
transport.proxyProtocolVersion = "v2"

[[proxies]]
name = "auth-yulinsec-cn-https"
type = "https"
localIP = "127.0.0.1"
localPort = 443
customDomains = ["auth.yulinsec.cn"]
transport.proxyProtocolVersion = "v2"

[[proxies]]
name = "outline-yulinsec-cn-http"
type = "http"
localIP = "127.0.0.1"
localPort = 80
customDomains = ["docs.yulinsec.cn"]
transport.proxyProtocolVersion = "v2"

[[proxies]]
name = "outline-yulinsec-cn-https"
type = "https"
localIP = "127.0.0.1"
localPort = 443
customDomains = ["docs.yulinsec.cn"]
transport.proxyProtocolVersion = "v2"

[[proxies]]
name = "portainer-yulinsec-cn-http"
type = "http"
localIP = "127.0.0.1"
localPort = 80
customDomains = ["portainer.yulinsec.cn"]
transport.proxyProtocolVersion = "v2"

[[proxies]]
name = "portainer-yulinsec-cn-https"
type = "https"
localIP = "127.0.0.1"
localPort = 443
customDomains = ["portainer.yulinsec.cn"]
transport.proxyProtocolVersion = "v2"

[[proxies]]
name = "wiki-yulinsec-cn-http"
type = "http"
localIP = "127.0.0.1"
localPort = 80
customDomains = ["wiki.yulinsec.cn"]
transport.proxyProtocolVersion = "v2"

[[proxies]]
name = "wiki-yulinsec-cn-https"
type = "https"
localIP = "127.0.0.1"
localPort = 443
customDomains = ["wiki.yulinsec.cn"]
transport.proxyProtocolVersion = "v2"

[[proxies]]
name = "grist-yulinsec-cn-http"
type = "http"
localIP = "127.0.0.1"
localPort = 80
customDomains = ["grist.yulinsec.cn"]
transport.proxyProtocolVersion = "v2"

[[proxies]]
name = "grist-yulinsec-cn-https"
type = "https"
localIP = "127.0.0.1"
localPort = 443
customDomains = ["grist.yulinsec.cn"]
transport.proxyProtocolVersion = "v2"

[[proxies]]
name = "homepage-yulinsec-cn-http"
type = "http"
localIP = "127.0.0.1"
localPort = 80
customDomains = ["homepage.yulinsec.cn"]
transport.proxyProtocolVersion = "v2"

[[proxies]]
name = "homepage-yulinsec-cn-https"
type = "https"
localIP = "127.0.0.1"
localPort = 443
customDomains = ["homepage.yulinsec.cn"]
transport.proxyProtocolVersion = "v2"
```

这里貌似冗余，后面处理

## 服务

### authentik

```yml
# /root/authentik/docker-compose.yml
services:
  postgresql:
    env_file:
    - .env
    environment:
      POSTGRES_DB: ${PG_DB:-authentik}
      POSTGRES_PASSWORD: ${PG_PASS:?database password required}
      POSTGRES_USER: ${PG_USER:-authentik}
    healthcheck:
      interval: 30s
      retries: 5
      start_period: 20s
      test:
      - CMD-SHELL
      - pg_isready -d $${POSTGRES_DB} -U $${POSTGRES_USER}
      timeout: 5s
    image: docker.io/library/postgres:16-alpine
    restart: unless-stopped
    volumes:
    - database:/var/lib/postgresql/data
  server:
    command: server
    depends_on:
      postgresql:
        condition: service_healthy
    env_file:
    - .env
    environment:
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB:-authentik}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER:-authentik}
      AUTHENTIK_SECRET_KEY: ${AUTHENTIK_SECRET_KEY:?secret key required}
    image: ${AUTHENTIK_IMAGE:-ghcr.io/goauthentik/server}:${AUTHENTIK_TAG:-2025.10.2}
    ports:
    - ${COMPOSE_PORT_HTTP:-9000}:9000
    - ${COMPOSE_PORT_HTTPS:-9443}:9443
    restart: unless-stopped
    volumes:
    - ./media:/media
    - ./custom-templates:/templates
  worker:
    command: worker
    depends_on:
      postgresql:
        condition: service_healthy
    env_file:
    - .env
    environment:
      AUTHENTIK_POSTGRESQL__HOST: postgresql
      AUTHENTIK_POSTGRESQL__NAME: ${PG_DB:-authentik}
      AUTHENTIK_POSTGRESQL__PASSWORD: ${PG_PASS}
      AUTHENTIK_POSTGRESQL__USER: ${PG_USER:-authentik}
      AUTHENTIK_SECRET_KEY: ${AUTHENTIK_SECRET_KEY:?secret key required}
    image: ${AUTHENTIK_IMAGE:-ghcr.io/goauthentik/server}:${AUTHENTIK_TAG:-2025.10.2}
    restart: unless-stopped
    user: root
    volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    - ./media:/media
    - ./certs:/certs
    - ./custom-templates:/templates
volumes:
  database:
    driver: local
```

### outline

```yml
# /root/outline/docker-compose.yml
services:
  outline:
    image: outlinewiki/outline:1.1.0
    env_file: ./docker.env
    ports:
      - "3030:3030"
    expose:
      - "3030"
    volumes:
      - ./data:/var/lib/outline/data
    depends_on:
      - postgres
      - redis

  redis:
    image: redis
    env_file: ./docker.env
    expose:
      - "6379"
    volumes:
      - ./redis.conf:/redis.conf
    command: ["redis-server", "/redis.conf"]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 30s
      retries: 3

  postgres:
    image: postgres:18
    env_file: ./docker.env
    expose:
      - "5432"
    volumes:
      - ./database:/var/lib/postgresql
    healthcheck:
      test: ["CMD", "pg_isready", "-d", "outline", "-U", "user"]
      interval: 30s
      timeout: 20s
      retries: 3
    environment:
      POSTGRES_USER: 'user'
      POSTGRES_PASSWORD: 'pass'
      POSTGRES_DB: 'outline'
```

```yml
# /root/outline/docker.env
# 汉化了，原文是英文
NODE_ENV=production

# 该 URL 应指向完全限定的、可公开访问的地址。若使用代理，此处为代理的 URL
URL=https://docs.yulinsec.cn

# Outline 服务器暴露的端口，需与 docker-compose.yml 中配置的端口一致
PORT=3030

# 有关运行独立协作服务器的说明，请查看文档(SERVICES.md)，常规运行时无需设置此项
COLLABORATION_URL=

# 若使用 Cloudfront/Cloudflare 等分发网络，可在此处配置。
# 配置后，JavaScript、样式表和图片的路径会更新为 CDN_URL 中定义的主机名。
# CDN 配置中的源服务器需与 URL 保持一致
CDN_URL=

# 要生成的进程数量。合理的估算规则是：服务器可用内存 ÷ 512（单位：MB）
WEB_CONCURRENCY=4

# 生成一个十六进制编码的 32 字节随机密钥。可在终端执行 `openssl rand -hex 32` 生成随机值
SECRET_KEY=...

# 生成一个唯一的随机密钥。格式无严格要求，也可通过终端 `openssl rand -hex 32` 生成
UTILS_SECRET=...

# 默认界面语言。可在 translate.getoutline.com 查看支持的语言代码及翻译完成度
DEFAULT_LANGUAGE=zh_CN


# ––––––––––––––––––––––––––––––––––––––
# ––––––––––––– 数据库配置 –––––––––––––
# ––––––––––––––––––––––––––––––––––––––

# 生产环境数据库的 URL，包含用户名、密码和数据库名
DATABASE_URL=postgres://user:pass@postgres:5432/outline

# 每个进程的内存数据库连接池设置。确保连接池大小不超过数据库允许的最大连接数
# 默认值为最小 0、最大 5
DATABASE_CONNECTION_POOL_MIN=
DATABASE_CONNECTION_POOL_MAX=

# 若连接 Postgres 不使用 SSL，可取消注释此行。仅当数据库与应用在同一台机器时适用
PGSSLMODE=disable


# ––––––––––––––––––––––––––––––––––––––
# ––––––––––––– Redis 配置 –––––––––––––
# ––––––––––––––––––––––––––––––––––––––

# 环境的 Redis URL，可指定兼容 ioredis 的 URL 或 Base64 编码的配置对象
# 文档：https://docs.getoutline.com/s/hosting/doc/redis-LGM4BFXYp4
REDIS_URL=redis://redis:6379


# ––––––––––––––––––––––––––––––––––––––
# ––––––––––– 文件存储配置 –––––––––––
# ––––––––––––––––––––––––––––––––––––––

# 指定存储系统类型，可选值为 "s3" 或 "local"
# - "local"：图片和文档附件存储在本地磁盘
# - "s3"：存储在兼容 S3 协议的网络存储服务
# 文档：https://docs.getoutline.com/s/hosting/doc/file-storage-N4M0T6Ypu7
FILE_STORAGE=local

# 若 FILE_STORAGE 配置为 "local"，此项设置所有附件/图片的存储上级目录
# 确保进程拥有该路径的创建权限和文件写入权限
FILE_STORAGE_LOCAL_ROOT_DIR=/var/lib/outline/data

# 上传附件的最大允许大小（单位：字节）
FILE_STORAGE_UPLOAD_MAX_SIZE=262144000

# 覆盖文档导入的最大大小，通常应小于文档附件的最大大小
FILE_STORAGE_IMPORT_MAX_SIZE=

# 覆盖工作区导入的最大大小，工作区导入文件可能特别大，且为临时文件会被自动定期删除
FILE_STORAGE_WORKSPACE_IMPORT_MAX_SIZE=

# 若在分布式架构中支持头像和文档附件的图片上传，且 FILE_STORAGE=s3 时，
# 需配置兼容 S3 的存储服务
AWS_ACCESS_KEY_ID=get_a_key_from_aws
AWS_SECRET_ACCESS_KEY=get_the_secret_of_above_key
AWS_REGION=xx-xxxx-x

# ––––––––––––––––––––––––––––––––––––––
# ––––––––––– SSL 配置 –––––––––––
# ––––––––––––––––––––––––––––––––––––––

# HTTPS 终止的 Base64 编码私钥和证书。这是三种 SSL 配置方式之一，可留空
# 文档：https://docs.getoutline.com/s/hosting/doc/ssl-pzk7WO8d1n
SSL_KEY=
SSL_CERT=

# 生产环境中自动重定向到 HTTPS。默认值为 true，若确定 SSL 在外部负载均衡器终止，可设为 false
FORCE_HTTPS=false


# ––––––––––––––––––––––––––––––––––––––
# ––––––––––– 身份认证配置 –––––––––––
# ––––––––––––––––––––––––––––––––––––––

# 第三方登录凭证，至少需要配置 Google、Slack、Discord 或 Microsoft 中的一种，
# 否则将无登录选项可用

# Slack 登录提供商
# 文档：https://docs.getoutline.com/s/hosting/doc/slack-sgMujR8J9J
SLACK_CLIENT_ID=
SLACK_CLIENT_SECRET=

# Google 登录提供商
# 文档：https://docs.getoutline.com/s/hosting/doc/google-hOuvtCmTqQ
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# Microsoft Entra / Azure AD 登录提供商
# 文档：https://docs.getoutline.com/s/hosting/doc/microsoft-entra-UVz6jsIOcv
AZURE_CLIENT_ID=
AZURE_CLIENT_SECRET=
AZURE_RESOURCE_APP_ID=

# Discord 登录提供商
# 文档：https://docs.getoutline.com/s/hosting/doc/discord-g4JdWFFub6
DISCORD_CLIENT_ID=
DISCORD_CLIENT_SECRET=
DISCORD_SERVER_ID=
DISCORD_SERVER_ROLES=

# 通用 OIDC 提供商
# 文档：https://docs.getoutline.com/s/hosting/doc/oidc-8CPBm6uC0I
OIDC_CLIENT_ID=...
OIDC_CLIENT_SECRET=...
OIDC_AUTH_URI=https://auth.yulinsec.cn/application/o/authorize/
OIDC_TOKEN_URI=https://auth.yulinsec.cn/application/o/token/
OIDC_USERINFO_URI=https://auth.yulinsec.cn/application/o/userinfo/
OIDC_LOGOUT_URI=https://auth.yulinsec.cn/application/o/outline/end-session/

# 指定从哪些声明中提取用户信息，支持 JWT 载荷中的任意有效 JSON 路径
OIDC_USERNAME_CLAIM=profile

# OIDC 认证的显示名称
OIDC_DISPLAY_NAME=YulinSec Auth

# 空格分隔的认证作用域
OIDC_SCOPES=openid profile email


# ––––––––––––––––––––––––––––––––––––––
# ––––––––––– 邮件配置 –––––––––––
# ––––––––––––––––––––––––––––––––––––––

# 要支持发送事务性邮件（如“文档已更新”或邮件登录），需连接 SMTP 服务器。
# 可配置的服务列表：https://community.nodemailer.com/2-0-0-beta/setup-smtp/well-known-services/
# 文档：https://docs.getoutline.com/s/hosting/doc/smtp-cqCJyZGMIB
SMTP_SERVICE=
SMTP_USERNAME=
SMTP_PASSWORD=
SMTP_FROM_EMAIL=


# ––––––––––––––––––––––––––––––––––––––
# ––––––––––– 限流配置 –––––––––––
# ––––––––––––––––––––––––––––––––––––––

# 是否启用限流功能
RATE_LIMITER_ENABLED=true

# 各个端点有硬编码的限流规则（需上述开关启用），此项为所有请求的全局限流
RATE_LIMITER_REQUESTS=1000
RATE_LIMITER_DURATION_WINDOW=60


# ––––––––––––––––––––––––––––––––––––––
# ––––––––––– 集成配置 –––––––––––
# ––––––––––––––––––––––––––––––––––––––

# GitHub 集成：支持预览 Issues 和 Pull Request 链接
# 文档：https://docs.getoutline.com/s/hosting/doc/github-GchT3NNxI9
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
GITHUB_WEBHOOK_SECRET=
GITHUB_APP_NAME=
GITHUB_APP_ID=
GITHUB_APP_PRIVATE_KEY=

# Linear 集成：支持将 Issue 链接预览为富文本提及
LINEAR_CLIENT_ID=
LINEAR_CLIENT_SECRET=

# Slack 完整集成（含搜索和频道发帖）：除了 Slack 认证外，还需配置以下项
# 文档：https://docs.getoutline.com/s/hosting/doc/slack-G2mc8DOJHk
SLACK_VERIFICATION_TOKEN=
SLACK_APP_ID=
SLACK_MESSAGE_ACTIONS=

# Dropbox 集成：按以下说明获取密钥 https://www.dropbox.com/developers/embedder#setup
# 并在应用设置中白名单你的域名
DROPBOX_APP_KEY=

# 可选启用 Sentry (sentry.io) 跟踪错误和性能
# 文档：https://docs.getoutline.com/s/hosting/doc/sentry-jxcFttcDl5
SENTRY_DSN=
SENTRY_TUNNEL=

# 启用从 Notion 工作区导入页面
# 文档：https://docs.getoutline.com/s/hosting/doc/notion-2v6g7WY3l3
NOTION_CLIENT_ID=
NOTION_CLIENT_SECRET=

# Iframely 集成：支持在 Outline 中预览第三方内容（如悬停外部链接显示预览）
# 文档：https://docs.getoutline.com/s/hosting/doc/iframely-HwLF1EZ9mo
IFRAMELY_URL=
IFRAMELY_API_KEY=


# ––––––––––––––––––––––––––––––––––––––
# ––––––––––– 调试配置 –––––––––––
# ––––––––––––––––––––––––––––––––––––––

# 是否允许安装程序通过发送匿名统计信息检查更新
ENABLE_UPDATES=true

# 要启用的调试类别，若你的代理已记录入站 HTTP 请求，可移除默认的 "http" 避免重复日志
DEBUG=http

# 配置服务器日志的最低严重级别，可选值：
# error, warn, info, http, verbose, debug, silly
LOG_LEVEL=info
```

### grist

```yml
# /root/grist/docker-compose.yml
services:
  grist:
    image: gristlabs/grist:latest
    environment:
      # Postgres database setup
      TYPEORM_DATABASE: grist
      TYPEORM_USERNAME: grist
      TYPEORM_HOST: grist-db
      TYPEORM_LOGGING: false
      TYPEORM_PASSWORD: ${DATABASE_PASSWORD}
      TYPEORM_PORT: 5432
      TYPEORM_TYPE: postgres

	  # OIDC核心配置
      GRIST_OIDC_IDP_ISSUER: "https://auth.yulinsec.cn/application/o/grist/.well-known/openid-configuration"
      # 替换为Authentik中Grist应用的Client ID
      GRIST_OIDC_IDP_CLIENT_ID: "..."
      # 替换为Authentik中Grist应用的Client Secret
      GRIST_OIDC_IDP_CLIENT_SECRET: "..."
      # Grist的外部访问地址（需与Authentik的Redirect URI一致）
      GRIST_OIDC_SP_HOST: "https://grist.yulinsec.cn"
      # 可选：请求的OIDC Scope（默认openid email profile）
      GRIST_OIDC_IDP_SCOPES: "openid email profile"
      # 可选：登出时跳过IDP的注销端点（若Authentik不支持RP-Initiated Logout）
      # GRIST_OIDC_IDP_SKIP_END_SESSION_ENDPOINT: "true"
      # 可选：用户名称/邮箱的属性映射（默认自动取name/email）
      GRIST_OIDC_SP_PROFILE_NAME_ATTR: "name"
      GRIST_OIDC_SP_PROFILE_EMAIL_ATTR: "email"
      GRIST_OIDC_SP_IGNORE_EMAIL_VERIFIED: true
      # Redis setup
      REDIS_URL: redis://grist-redis

      # MinIO setup. This requires the bucket set up on the MinIO instance with versioning enabled.
      GRIST_DOCS_MINIO_ACCESS_KEY: grist
      GRIST_DOCS_MINIO_SECRET_KEY: ${MINIO_PASSWORD}
      GRIST_DOCS_MINIO_USE_SSL: 0
      GRIST_DOCS_MINIO_BUCKET: grist-docs
      GRIST_DOCS_MINIO_ENDPOINT: grist-minio
      GRIST_DOCS_MINIO_PORT: 9000

      # 新增核心配置
      GRIST_HOME_URL: ${GRIST_EXTERNAL_URL}  # 外部访问地址，OIDC回调依赖
      GRIST_SINGLE_ORG: "YulinSec"            # 单组织名称，自定义
      GRIST_PORT: 8484                        # 容器内端口（固定）
      NODE_ENV: production                    # 生产环境
    volumes:
      # Where to store persistent data, such as documents.
      - ${PERSIST_DIR}/grist:/persist
    ports:
      - 8484:8484
    depends_on:
      - grist-db
      - grist-redis
      - grist-minio
      - minio-setup

  grist-db:
    image: postgres:alpine
    environment:
        POSTGRES_DB: grist
        POSTGRES_USER: grist
        POSTGRES_PASSWORD: ${DATABASE_PASSWORD}
    volumes:
      - ${PERSIST_DIR}/postgres:/var/lib/postgresql/data

  grist-redis:
    image: redis:alpine
    volumes:
      - ${PERSIST_DIR}/redis:/data

  grist-minio:
    image: minio/minio:latest
    environment:
      MINIO_ROOT_USER: grist
      MINIO_ROOT_PASSWORD: ${MINIO_PASSWORD}
    volumes:
      - ${PERSIST_DIR}/minio:/data
    command:
      server /data --console-address=":9001"

  # This sets up the buckets required in MinIO. It is only needed to make this example work.
  # It isn't necessary for deployment and can be safely removed.
  minio-setup:
    image: minio/mc
    environment:
      MINIO_PASSWORD: ${MINIO_PASSWORD}
    depends_on:
      grist-minio:
        condition: service_started
    restart: on-failure
    entrypoint: >
      /bin/sh -c "
      /usr/bin/mc alias set myminio http://grist-minio:9000 grist '$MINIO_PASSWORD';
      /usr/bin/mc mb myminio/grist-docs;
      /usr/bin/mc anonymous set public myminio/grist-docs;
      /usr/bin/mc version enable myminio/grist-docs;
      "
```

```yml
# /root/grist/.env
# 数据库密码
DATABASE_PASSWORD=...
# MinIO根密码（需8位以上）
MINIO_PASSWORD=...
# 持久化数据目录（宿主机路径）
PERSIST_DIR=/root/grist/data
# Grist外部访问地址（核心：OIDC回调依赖此地址）
GRIST_EXTERNAL_URL=https://grist.yulinsec.cn
```

todo:同步脚本

### homepage

```yml
# /root/homepage
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    ports:
      - 3000:3000
    volumes:
      - /path/to/config:/app/config # Make sure your local config directory exists
      - /var/run/docker.sock:/var/run/docker.sock # (optional) For docker integrations
    environment:
      HOMEPAGE_ALLOWED_HOSTS: gethomepage.dev # required, may need port. See gethomepage.dev/installation/#homepage_allowed_hosts
```

### WireGuard

```yml
volumes:
  etc_wireguard:

services:
  wg-easy:
    #environment:
    #  Optional:
    #  - PORT=51821
    #  - HOST=0.0.0.0
    #  - INSECURE=false

    image: ghcr.io/wg-easy/wg-easy:15
    container_name: wg-easy
    networks:
      wg:
        ipv4_address: 10.42.42.42
        ipv6_address: fdcc:ad94:bacf:61a3::2a
    volumes:
      - etc_wireguard:/etc/wireguard
      - /lib/modules:/lib/modules:ro
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
      # - NET_RAW # ⚠️ Uncomment if using Podman
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv6.conf.all.disable_ipv6=0
      - net.ipv6.conf.all.forwarding=1
      - net.ipv6.conf.default.forwarding=1

networks:
  wg:
    driver: bridge
    enable_ipv6: true
    ipam:
      driver: default
      config:
        - subnet: 10.42.42.0/24
        - subnet: fdcc:ad94:bacf:61a3::/64
```
