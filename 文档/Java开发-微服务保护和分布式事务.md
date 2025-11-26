---
title: "微服务保护及分布式事务"
subtitle: "记录微服务保护及分布式事务的学习"
summary: "记录微服务保护及分布式事务的学习"
date: 2025-09-08
lastmod: 2025-09-08
draft: false
toc:
 enable: true
weight: false
categories: ["开发"]
tags: ["技术"]
---

## 微服务保护

问题：

- 雪崩问题：某个服务故障，导致整个链路失效
  - 微服务相互调用
  - 没有做好异常处理
  - 所有服务级联失败

解决思路：

- 尽量避免服务故障、阻塞

- 做好异常的后备方案

方案：

- 请求限流
- 线程隔离：控制业务可用线程数量
- 服务熔断：将异常比例过高的接口断开，直接走 fallback
- 失败处理：定义 fallback 处理逻辑

### Sentinel

整合到微服务中，配置控制台

簇点链路：默认情况下，Sentinel 拦截的只是 Controller 的请求路径，故需要配置其拦截请求方法。

#### 请求限流

设置 **QPS**，每秒最多请求线程数

**Jmeter**

请求模拟工具，用于测试压力

#### 线程隔离

服务 B 出现阻塞或故障时，调用服务 B 的服务 A 的资源也可能因此被耗尽，故必须限制服务 A 中调用服务B的线程数。保护服务 A 中其他接口。

#### Fallback

将 FeignClient 添加到服务，若超限，则调用其中的 FallFactory 的接口。

#### 服务熔断

解决雪崩问题的重要手段。有断路器统计服务调用的异常比例、慢请求比例，若超出阈值则熔断改服务。

## 分布式事务

分布式系统中，一个业务需要多个服务共同完成，则这多个服务需要同时成功或失败。

解决思路：

- 各个子事务之间能感知到彼此的状态

### Seata

#### Seata架构

TC：事务协调者，协调全局事务提交或回滚

TM：事务管理器，定义全局事务的范围，开始提交或回滚（入口）

RM：资源管理器，与 TC 交谈以注册事务状态

#### 部署 TC 服务

seata 用 docker 部署，然后注册到 Nacos 上

#### 微服务集成 Seata

在 `application.yml` 中添加配置，让微服务找到 TC 地址

抽取共享配置到 nacos、划分事务组

```yaml
seata:
  registry: # TC服务注册中心的配置，微服务根据这些信息去注册中心获取tc服务地址
    type: nacos # 注册中心类型 nacos
    nacos:
      server-addr: 192.168.150.101:8848 # nacos地址
      namespace: "" # namespace，默认为空
      group: DEFAULT_GROUP # 分组，默认是DEFAULT_GROUP
      application: seata-server # seata服务名称
      username: nacos
      password: nacos
  tx-service-group: hmall # 事务组名称
  service:
    vgroup-mapping: # 事务组与tc集群的映射关系
      hmall: "default"
```

#### XA 模式

步骤：

- RM 注册分支事务到 TC
- RM 执行 sql 但不提交
- RM 报告执行状态到 TC
- TC 检查各分支执行状态，RM 等待 TC 指令

问题：

- 需要锁定数据库资源，需要等待，性能较差

#### AT 模式

弥补 XA 模式中资源锁定周期过长的缺陷

步骤：

- 注册分支事务
- 记录数据快照
- 执行 sql 并提交
- 报告事务状态
- 删除快照 / 根据快照恢复数据

问题：

- 短暂的数据不一致

使用：

- 对每个服务创建一个 `undo_log` 表