---
title: "PVE的lxc环境下搭建grist表格"
subtitle: "PVE的lxc环境下搭建grist表格"
summary: "复盘一下有点曲折的搭建过程"
description: "复盘一下有点曲折的搭建过程"
image: ""
date: 2025-11-20
lastmod: 2025-11-20
draft: false
toc:
 enable: true
hiddenFromHomePage: false
weight: false
categories: ["DevOps"]
tags: ["技术杂项"]
---

## 准备工作

grist 的 github：[Grist is the evolution of spreadsheets.](https://github.com/gristlabs/grist-core)

先要创建一个虚拟机，选用了 lxc 而非 vm，其实是一个不太好的选择，导致在搭建过程中磕磕绊绊。应该是因为 PVE 对 lxc 进行了比较多的限制。

![image-20251120212038256](https://raw.githubusercontent.com/calendar0917/images/master/image-20251120212038256.png)

正常创建就可以了，镜像选的是 ubuntu24.04。好像可以用模板？但是不知道密码，登不进去……

## 配置 docker 环境

从头开始配还是有点麻烦，主要原因是镜像源太不稳定。先配置 apt 的源，把 docker 下下来，再配置 docker 的镜像，多找几个试试。还弄了 clash，但是没有用上（docker 的代理还要另外配置）。

具体内容添加到了：[配置docker](https://calendar0917.github.io/posts/技术杂项-配置docker/)

## 配置 grist

官方 git 里面有 dockers-compose-examples：

![image-20251120212528238](https://raw.githubusercontent.com/calendar0917/images/master/image-20251120212528238.png)

一开始用的是 grist-traefik（反向代理工具）-basic-auth 的，弄完了才知道出外网没有域名，反向代理很鸡肋……接着就改成了 [grist-with-postgres-redis-minio](https://github.com/gristlabs/grist-core/tree/main/docker-compose-examples/grist-with-postgres-redis-minio)，有了基础的存储，但是没有登录认证。

配置文件如下：

```yml
services:
  grist:
    image: gristlabs/grist:latest
    environment:
      TYPEORM_DATABASE: grist
      TYPEORM_USERNAME: grist
      TYPEORM_HOST: grist-db
      TYPEORM_LOGGING: false
      TYPEORM_PASSWORD: 12345678
      TYPEORM_PORT: 5432
      TYPEORM_TYPE: postgres
      REDIS_URL: redis://grist-redis
      GRIST_DOCS_MINIO_ACCESS_KEY: grist
      GRIST_DOCS_MINIO_SECRET_KEY: 12345678
      GRIST_DOCS_MINIO_USE_SSL: 0
      GRIST_DOCS_MINIO_BUCKET: grist-docs
      GRIST_DOCS_MINIO_ENDPOINT: grist-minio
      GRIST_DOCS_MINIO_PORT: 9000
    volumes:
      - grist-persist:/persist  # 命名卷
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
        POSTGRES_PASSWORD: 12345678
    volumes:
      - grist-postgres:/var/lib/postgresql/data  # 命名卷

  grist-redis:
    image: redis:alpine
    volumes:
      - grist-redis:/data  # 命名卷

  grist-minio:
    image: minio/minio:latest
    environment:
      MINIO_ROOT_USER: grist
      MINIO_ROOT_PASSWORD: 12345678
    volumes:
      - grist-minio:/data  # 命名卷
    command:
      server /data --console-address=":9001"

  minio-setup:
    image: minio/mc
    environment:
      MINIO_PASSWORD: 12345678
    depends_on:
      grist-minio:
        condition: service_started
    restart: on-failure
    entrypoint: >
      /bin/sh -c "
      /usr/bin/mc alias set myminio http://grist-minio:9000 grist '12345678';
      /usr/bin/mc mb myminio/grist-docs;
      /usr/bin/mc anonymous set public myminio/grist-docs;
      /usr/bin/mc version enable myminio/grist-docs;
      "

# 定义命名卷（Docker自动创建，存储在/var/lib/docker/volumes）
volumes:
  grist-persist:
  grist-postgres:
  grist-redis:
  grist-minio:
```

这里把路径、密码写死了，读取`.env`的话好像有点问题。

### 要注意的

在 lxc 中，对权限会有限制，所以

- docker 的版本有要求！参见：https://forum.proxmox.com/threads/docker-inside-lxc-net-ipv4-ip_unprivileged_port_start-error.175437/

- 对挂载的数据卷有要求！
  - 不能挂载到 `/root` 下
  - 后面改成了自定义命名卷，见上

## 穿透

用了 frp，还是弄了好一会

```toml
# frpc.toml 客户端配置
[common]
server_addr =8.137.38.223
server_port = 7000            # 服务端bind_port
token = 11111

# Grist端口映射规则
[[proxies]]
name = "grist-proxy"
type = "tcp"
local_ip = "127.0.0.1"
local_port = 8484
remote_port = 9000
```

```toml
# frps.toml 服务端配置
[common]
bind_addr = 0.0.0.0 # 指定监听 ipv4
bind_port = 7000  # TOML格式中，frp的配置项是下划线分隔（bind_port），不是驼峰（bindPort）！
token = 11111
```

分别运行`nohup ./frps -c ./frps.toml > frps.log &`即可。

> 写下来才感觉没什么，做的时候思路就很乱。各种基础配置没有一个模板，就一直到处找，很低效。
>
> 对 docker、反向代理的理解也是模棱两可，导致实际操作起来没有方向，不知道哪里出了问题、要怎么解决。
>
> 然后具体的服务配置其实已经在 yml 里写得很清楚了，但是自己改的能力还是很差。以及，vim 要学起来用了……