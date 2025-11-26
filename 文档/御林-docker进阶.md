---
title: "御林招新题：docker 进阶"
subtitle: "御林招新题：docker 进阶"
summary: "学习 docker 的进阶知识"
description: "学习 docker 的进阶知识"
image: ""
date: 2025-10-19
lastmod: 2025-10-19
draft: false
hiddenFromHomePage: True
toc:
 enable: true
weight: false
categories: ["CTF"]
tags: ["CTF"]
---

## **多阶段构建（Multi-stage build）**

- **目标**：优化你的镜像大小，只包含运行应用所需的最小组件。

> - 什么是多阶段构建？
>   - 允许在一个 `Dockerfile` 中定义多个构建阶段，每个阶段可以使用不同的基础镜像，最终只将必要的文件复制到最终镜像中，从而剔除构建过程中产生的冗余内容（如编译工具、临时文件、开发依赖等）。
>   - **构建阶段**：使用包含编译 / 打包工具的镜像，完成代码编译、依赖安装等操作；
>   - **运行阶段**：使用轻量级基础镜像（如 `alpine`），仅复制构建阶段的产物（如可执行文件、Jar 包），最终镜像只包含运行所需的最小环境。
>
> ```dockfile
> # 第一阶段：构建阶段（可命名）
> FROM 基础镜像1 AS 阶段名1
> # 执行构建操作（如编译、安装依赖）
> RUN 命令1
> COPY 源码 目标路径
> 
> # 第二阶段：运行阶段（最终镜像）
> FROM 基础镜像2 AS 阶段名2
> # 从第一阶段复制构建产物
> COPY --from=阶段名1 构建阶段的产物路径 最终镜像的目标路径
> # 定义运行命令
> CMD ["启动命令"]
> ```
>
> - 第一阶段去哪里了？
>   - 第一阶段的产物，要么被**主动复制到最终镜像**（成为运行时必需的部分），要么作为**中间层被 Docker 缓存**（用于加速后续构建，但不进入最终镜像）。

- **挑战**：
[[编写Dockerfile]]
  - 编写一个 `Dockerfile`，使用**多阶段构建**来打包一个简单的 Web 应用（例如，一个基于 Python Flask 的应用）。

  ```py
  from flask import Flask
  
  app = Flask(__name__)
  
  @app.route('/')
  def hello():
      return 'Hello from Flask!'
  
  if __name__ == '__main__':
      app.run(host='0.0.0.0', port=5000)
  ```

  - 在第一阶段，使用完整的开发镜像（例如 `python:3.9`）来安装依赖并构建应用。

  - 在第二阶段，使用一个轻量级的运行时镜像（例如 `python:3.9-slim` 或 `alpine`）作为基础，只将第一阶段构建好的应用代码和必要的依赖文件复制过来。

  ```dockerfile
  # 第一阶段：构建阶段，使用完整的开发镜像
  FROM python:3.9 as builder
  
  # 设置工作目录
  WORKDIR /app
  
  # 复制依赖文件并安装
  COPY requirements.txt .
  RUN pip install -r requirements.txt
  
  # 复制应用代码
  COPY app.py .
  
  # 第二阶段：运行阶段，使用轻量级的运行时镜像
  FROM python:3.9-slim
  
  # 设置工作目录
  WORKDIR /app
  
  # 从构建阶段复制必要的文件（安装好的依赖和应用代码）
  COPY --from=builder /app/requirements.txt .
  COPY --from=builder /app/app.py .
  COPY --from=builder /usr/local/lib/python3.9/site-packages/ /usr/local/lib/python3.9/site-packages/
  
  # 暴露应用端口
  EXPOSE 5000
  
  # 启动应用
  CMD ["python", "app.py"]
  ```

  ```dockerfile
  # 单阶段
  FROM python:3.9
  
  WORKDIR /app
  
  COPY requirements.txt .
  RUN pip install -r requirements.txt
  
  COPY app.py .
  
  EXPOSE 5000
  
  CMD ["python", "app.py"]
  ```

  - **验证**：分别构建一个单阶段镜像和一个多阶段镜像，并使用 `docker images` 命令比较它们的大小，说明多阶段构建的优势。

  ```shell
  [root@localhost docker_stage_2]# docker images
  REPOSITORY        TAG       IMAGE ID       CREATED          SIZE
  multi_stage       latest    e9c0c0a489a8   16 minutes ago   148MB
  docker_nostage    latest    033ed26b2d7f   16 minutes ago   1.1GB
  ```

  > 多阶段可以删去不必要的组件，精简了镜像的大小
  >
  > - 单阶段使用的 `python:3.9` 基于完整的 Debian 系统，多阶段最终使用的 `python:3.9-slim` 是精简版
  >
  > 为什么不能直接用 slim 构建？
  >
  > - 如果直接用 `slim` 镜像构建（单阶段），执行 `pip install -r requirements.txt` 时，若遇到需要编译的包，会因缺少 `gcc` 等工具而失败，报错类似：`error: command 'gcc' failed: No such file or directory`，所以需要构建后再复制编译后的模块
  >
  > 怎么知道所需要保留的包的路径？
  >
  > - 可以创建临时容器：`docker run -it --rm python:3.9 /bin/bash`，进入后 `python -m site` 或 `pip show flask | grep "Location"`
  > - 也可以先构建一个单阶段容器，在 dockerfile 中输出依赖路径，再作修改

## **镜像版本管理与标签（Tagging）**

- **目标**：为你的镜像打上清晰的版本标签，方便管理和追溯。

- **挑战**：

  - 为你上一步构建的多阶段镜像打上至少两个标签（例如 `your-app:1.0.0` 和 `your-app:latest`）。
  - 使用 `docker images` 命令验证标签是否正确应用。

  > `docker tag <ID> name:<tag>`

  ```shell
  [root@localhost docker_stage_2]# docker tag e9c0c0a489a8 multi:latest
  [root@localhost docker_stage_2]# docker images
  REPOSITORY        TAG       IMAGE ID       CREATED             SIZE
  multi             1.0.0     e9c0c0a489a8   29 minutes ago      148MB
  multi             latest    e9c0c0a489a8   29 minutes ago      148MB
  ```

## **镜像的打包与加载**

- **目标**：掌握在没有 Docker Registry 的情况下，迁移镜像的方法。

> `docker save -o your-app-1.0.0.tar your-app:1.0.0`

- **挑战**：

  - 使用 `docker save` 命令将你构建的镜像（`your-app:1.0.0`）打包成一个 `.tar` 文件。
  - 将该 `.tar` 文件复制到另一台机器（或在当前机器上删除本地镜像），然后使用 `docker load` 命令加载该 `.tar` 文件。

  > 删除镜像 `docker rmi your-app:1.0.0`
  >
  > 加载镜像 `docker load -i target`

  - **验证**：使用 `docker images` 命令，确认镜像已成功加载，并可以正常运行。

  > [root@localhost docker_stage_2]# docker load -i multi-1.0.0.tar
  > Loaded image: multi:1.0.0
  > [root@localhost docker_stage_2]# docker images
  > REPOSITORY        TAG       IMAGE ID       CREATED             SIZE
  > multi             1.0.0     e9c0c0a489a8   35 minutes ago      148MB

## **推送到私有仓库**

- **目标**：将你的镜像推送到一个私有的 Docker Registry，模拟团队协作环境。

- **挑战**：

  - 在本地运行一个临时的 Docker Registry 容器。
  - 使用 `docker tag` 命令为你的镜像打上指向该私有仓库的标签（例如 `localhost:5000/your-app:1.0.0`）。
  - 使用 `docker push` 命令将镜像推送到本地私有仓库。
  - **验证**：使用 `docker pull` 命令从该私有仓库拉取镜像，确认推送和拉取流程畅通。

  ```shell
  [root@localhost docker_stage_2]# docker rmi multi_stage:latest
  Untagged: multi_stage:latest
  Deleted: sha256:e9c0c0a489a800d997c60d99cfc4fd11b7416afdb10f80e5d1f5b68bfdf5a16b
  [root@localhost docker_stage_2]# docker load -i multi-1.0.0.tar
  Loaded image: multi:1.0.0
  [root@localhost docker_stage_2]# docker run -p 8001:5000 multi:1.0.0
  ```

  > 如果 rmi 的时候镜像有多个 tag，只会删除 tag，只有只剩一个 tag 时会彻底删除镜像

![image-20251019094027247](https://raw.githubusercontent.com/calendar0917/images/master/image-20251019094027247.png)

> - 属于一个 compose 的要如何一起 stop？
>   - 在 `docker-compose.yml` 文件所在的目录下，使用 **`docker compose stop` 命令**