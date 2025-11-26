---
title: "御林招新题：Homepage"
subtitle: "御林招新题：Homepage"
summary: "学习用 Homepage 制作工作台"
description: "学习用 Homepage 制作工作台"
image: "https://raw.githubusercontent.com/calendar0917/images/master/20251020172907999.png"
date: 2025-10-20
lastmod: 2025-10-20
hiddenFromHomePage: True
draft: false
toc:
 enable: true
weight: false
categories: ["CTF"]
tags: ["CTF"]
---

**任务：安装并配置一个基础的 Homepage**

> - 什么是 Homepage？
>
> > A modern, *fully static, fast*, secure *fully proxied*, highly customizable application dashboard with integrations for over 100 services and translations into multiple languages. Easily configured via YAML files or through docker label discovery.

## **阅读文档，选择安装方式**

- 访问 [Homepage 的官方 GitHub](https://github.com/gethomepage/homepage) 或官方网站。
- 在文档中找到“安装指南”部分，根据你已经学过的知识，选择最合适的安装方式（**推荐使用 Docker 或 Docker Compose**）。
- 根据文档说明，完成 Homepage 的安装。

## **理解配置，设置主页标题**

- 在文档中找到“配置”部分，了解其配置文件的结构。

```yml
# docker compose 的配置
# 可以用 docker proxy 来反向代理，增加安全性？
# 保存为 docker-compose.yml,然后 docker-compose up -d 启动,后续直接修改 yml,docker-compose restart 即可修改配置
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    environment:
      HOMEPAGE_ALLOWED_HOSTS: 8.137.38.223:3000 # 补上端口
      # required, may need port. See gethomepage.dev/installation/#homepage_allowed_hosts
      PUID: 1000 # optional, your user id
      PGID: 1000 # optional, your group id
    ports:
      - 3000:3000
    volumes:
      - ~/docker/homepage/config:/app/config # Make sure your local config directory exists
      # - /var/run/docker.sock:/var/run/docker.sock:ro # optional, for docker integrations # 用到 docker 集成的时候需要
    restart: unless-stopped
```

> mkdir -p ~/docker/homepage/config
>
> 启动容器后自动挂载
>
> - 安装 docker-compose
>   - ~~加装一下（要用 pip3），又发现要 rust 环境，再加装--~~
>   - `sudo curl -L "https://github.com/docker/compose/releases/download/1.29.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose`
>   - `sudo chmod +x /usr/local/bin/docker-compose`

![image-20251020091435920](https://raw.githubusercontent.com/calendar0917/images/master/image-20251020091435920.png)

进去了但没完全进去……

> ~~原因是配置文件只写了域名没写端口，改一下~~
>
> 原因是 HOMEPAGE_ALLOWED_HOSTS 得要配置，先设置为 “*” 禁用了（不安全，可能会被攻击）
>
> - restart 还没办法重载配置……需要docker-compose down然后docker-compose up -d，就可以进入了

![image-20251020093454972](https://raw.githubusercontent.com/calendar0917/images/master/image-20251020093454972.png)

```yml
# cofig 的配置
my-remote-docker:
  host: 192.168.0.101
  port: 2375
```



- 根据文档说明，修改配置文件，将你的主页标题设置为你喜欢的名字（例如：“我的DevOps控制台”）。

> 参考：[Settings - Homepage](https://gethomepage.dev/configs/settings/)，修改完可以热加载
>
> 有主题、外观等等

## **添加你的第一个服务**

- 在文档中找到“服务（Services）”或“应用程序（Apps）”相关的章节。

> 参考：[Services - Homepage](https://gethomepage.dev/configs/services/)，修改 service.yaml

- 根据文档中的示例，在配置文件中添加至少一个服务。这个服务可以是：
  - 你在 Docker 任务中搭建的 `searxng`。
  - 一个你常用的网站，例如 GitHub、你自己的博客或任何其他网站。
- **要求**：确保你正确配置了服务的名称、图标和URL。

![image-20251020095731270](https://raw.githubusercontent.com/calendar0917/images/master/image-20251020095731270.png)

```yaml
- Group One:
    - My Blog:
        icon: https://raw.githubusercontent.com/calendar0917/images/master/6a4b02385b1bb87d52812566164e8031.jpg
        href: https://calendar0917.github.io/
        description: Welcome To My Blog!

- Group Two:
    - 御林工作室:
        icon: http://recruit.yulinsec.cn/assets/favicon-CKa7I3RX.webp
        href: http://recruit.yulinsec.cn/#/game
        description: 来写几道题吧
```



## **验证**

- 重新加载或重启你的 Homepage 服务。
- 在浏览器中访问你的 Homepage，确认标题已更改，并且你添加的服务图标可以正常点击，并跳转到正确的页面。

>  见上

## **进阶服务与集成**

- 在文档中找到“增强服务（Enhanced services）”或“小部件（Widgets）”部分。
- 添加一个支持状态检查的服务。例如，你可以添加你的 `searxng` 服务，并配置 Homepage 能够自动检查其运行状态（在线/离线）。

```yml
# service.yaml
- Group Three:
    - Searxng:
        href: http://8.137.38.223:8081
        siteMonitor: http://8.137.38.223:8081
        icon: searxng.png
```

![image-20251020123744298](https://raw.githubusercontent.com/calendar0917/images/master/image-20251020123744298.png)

## **小部件定制**

- 在文档中找到“小部件”章节，根据说明在你的主页上添加至少一个实用的小部件。

> 先看看有什么 Widgets(小部件) 可以用：[Info Widgets - Homepage](https://gethomepage.dev/widgets/info/)

```yaml
- datetime:
    text_size: xl
    format:
      timeStyle: short
```

> 添加了一个时间

- 尝试添加一个 Docker 状态小部件，使其能显示你的 Docker 容器数量或状态，这需要你在文档中找到如何与 Docker API 集成的方法。

~~参考：[using-socket-directly](https://gethomepage.dev/configs/docker/#using-socket-directly)~~

> ~~需要用到 docker socket 集成，上面的 docker-compose.yml 注释要去掉了,还要把PGID、PUID删掉，才能以 root 运行（？）~~
>
> 不知道怎么使用，换方案

参考：[HomePage导航下集 常见组件的设置](https://www.bilibili.com/video/BV1xZ42127eJ/?spm_id_from=333.1007.top_right_bar_window_history.content.click&vd_source=f329e64c57c95b7adedc05b814353e29)，这里提到了 portainer 可视化工具，刚好 homepage 有整合

安装参考：[Docker | docker安装portainer详细步骤-CSDN博客](https://blog.csdn.net/weixin_44649780/article/details/128401975)

```shell
docker pull portainer/portainer-ce
# 启动镜像
docker run -d -p 9000:9000 -p 9443:9443 -v /var/run/docker.sock:/var/run/docker.sock -v /dockerData/portainer:/data --restart=always --name portainer portainer/portainer-ce:latest
```

```yaml
# setting.yaml
- Group Three:
    - Searxng:
        href: http://8.137.38.223:8081
        siteMonitor: http://8.137.38.223:8081
        icon: searxng.png

    - Portainer:
        icon: portainer.png
        href: http://8.137.38.223:9000 # Portainer IP:9000
        description: Portainer
        ping: 127.0.0.1 # Portainer IP
        server: my-docker
        showStats: true
        container: portainer
		widget:
          type: portainer
          url: https://8.137.38.223:9443
          env: 2
          key: ......

```

> - 遇到的问题：
>
>   刚开始没有映射 9443 端口（用于认证？）
>
>   - 重新映射只需要 docker rm ...，这样只会删除容器而不会删除数据卷，然后重新 docker run 即可
>
>   感觉暴露了 portainer 有点危险
>
>   - portainer 自带登录验证

效果：

![image-20251020165610688](https://raw.githubusercontent.com/calendar0917/images/master/image-20251020165610688.png)

## **主题与布局自定义**

- 在文档中找到“主题（Themes）”和“布局（Layout）”相关的章节。

> [Theme](https://gethomepage.dev/configs/settings/#theme)

- 尝试更改 Homepage 的默认主题，或者通过修改配置文件，调整服务图标的排列顺序或分组方式。

```yaml
# setting.yaml
layout:
  Group One:
    style: row
    columns: 2
```

Group One 由单列变为双列：

![image-20251020171307094](https://raw.githubusercontent.com/calendar0917/images/master/image-20251020171307094.png)

- **挑战**：尝试在配置文件中添加自定义 CSS，改变某个元素的颜色或字体。这需要你仔细阅读文档中关于“自定义”的部分。

> [Custom-css-js](https://gethomepage.dev/configs/custom-css-js/)，很简短的说明
>
> To add custom css simply edit the `custom.css` file under your config directory, similarly for javascript you would edit `custom.js`.

- 刚开始不知道去哪里找元素，其实直接 F12 看元素所属的类，然后修改就可以了

```css
<--! custom.css 字体变为红色 -->
.service-group-name{
    color: #FF3A00; 
}
```

效果：

![image-20251020172805821](https://raw.githubusercontent.com/calendar0917/images/master/image-20251020172805821.png)