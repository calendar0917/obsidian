---
title: "网关及配置中心"
subtitle: "网关及配置中心"
summary: "记录微服务-网关及配置中心的学习"
description: "记录微服务-网关及配置中心的学习"
date: 2025-10-01
lastmod: 2025-10-01
draft: false
toc:
 enable: true
weight: false
categories: ["开发"]
tags: ["技术"]
---

## 网关

### 介绍

网络的关口，负责请求的路由、转发、身份检验。分为阻塞式、响应式。微服务将服务注册到注册中心，网关进行服务拉取返回给前端。

### 使用

1. 创建新模块
2. 引入网关依赖
3. 编写启动类
4. 配置路由

```yml
spring:
  cloud:
    gateway:
      routes:
        - id: item # 路由规则id，自定义，唯一
          uri: lb://item-service # 路由目标微服务，lb代表负载均衡
          predicates: # 路由断言，判断请求是否符合规则，符合则路由到目标
            - Path=/items/** # 以请求路径做判断，以/items开头则符合
        - id: xx
          uri: lb://xx-service
          predicates:
            - Path=/xx/**
```

另有各种路由种类、路由过滤器。

### 登录校验

需要在网关转发之前进行校验，即添加过滤器。

网关底层流程：

1. HandlerMapping 路由映射器
2. WebHandler 请求处理器，即过滤器处理器
3. PRE（在这里实现） + POST 阶段

Q: 网关如何将用户信息传递给微服务？

- Http 传送 --> 用请求头

Q: 微服务之间如何传递用户信息？

#### 自定义过滤器

1. GatewayFilter：指定路由生效
2. GlobalFilter：全局过滤器，作用于所有路由

```java
@Component
public class MyGlobalFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 1.获取请求
        ServerHttpRequest request = exchange.getRequest();
        // 2.过滤器业务处理
        System.out.println("GlobalFilter pre阶段 执行了。");
        // 3.放行
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        // 过滤器执行顺序，值越小，优先级越高
        return 0;
    }
}
```

#### 网关拦截逻辑：

1. 获取 request
2. 根据 URL 判断是否需要拦截

3. 获取 Token，解析校验

#### 网关传递服务

1. 用 `ServerWebExchange` 类下的 API 来给请求头添加鉴权信息，再发送给后续服务。

2. 将登录检验封装为工具模块（配置类），统一扫描调用。
3. 配置类的配置：`@ConditionalOnClass(DispatcherServlet.class)` 只拦截到后端 SpringMVC 的请求（否则其他模块扫描不到），绕过网关。

#### 微服务间传递信息

利用 OpenFeign 的 `RequestTemplate` 类更改请求头传递，保存请求头。

## 配置管理中心

问题：

- 微服务重复配置过多，维护成本高
- 更改配置不方便，需要重启服务、网关

解决：

- 通过配置管理实现热更新、配置共享

### 配置管理

#### NACOS 可视化编辑共享配置

包含：

1. 数据库
2. 日志
3. Swagger

4. ……

#### 微服务拉取共享配置

流程：

1. 启动，加载 `bootstrap` 引导类
2. 拉取 Nacos 配置
3. 初始化 `ApplicationContext`上下文
4. 加载 `application.yml` ，拉取共享配置，合并配置

#### 配置热更新

前提条件

1. nacos 中要有于微服务名有关的配置文件
2. 微服务中要以特定方式读取需要热更新的配置属性

#### 动态路由

要求：

- 监听 Nacos 配置变更信息

```java
private final NacosConfigManager nacosConfigManager;

public void initRouteConfigListener() throws NacosException {
    // 1.注册监听器并首次拉取配置
    String configInfo = nacosConfigManager.getConfigService()
            .getConfigAndSignListener(dataId, group, 5000, new Listener() {
                @Override
                public Executor getExecutor() {
                    return null;
                }

                @Override
                public void receiveConfigInfo(String configInfo) {
                    // TODO 监听到配置变更，更新一次配置
                }
            });
    // TODO 2.首次启动时，更新一次配置
}
再定义 UpdateConfigInfo(),删除旧的路由、重新读取新路由
```

