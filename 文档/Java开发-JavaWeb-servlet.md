---
title: "JavaWeb-Servlet"
subtitle: "JavaWeb Servlet 的学习笔记"
description: "JavaWeb Servlet 的学习笔记，包含MVC的优化"
summary: "JavaWeb Servlet 的学习笔记"
date: 2025-10-09
lastmod: 2025-10-09
draft: false
toc:
 enable: true
weight: false
categories: ["开发"]
tags: ["技术"]
---

## Tomcat

服务器容器，将服务部署（deploy）到容器内

目录结构：

- bin 可执行文件目录
- conf 配置文件目录
- webapps 项目部署的目录
  - 项目内容存放到 webapps-name-WEBINF 文件夹下，即可访问
- work 工作目录

## Servlet

### 获取参数

1. 在 web.xml 中定义 `servlet-mapping`，指定接收 url 请求对应的类
   1. 现在支持注解注册 `@WebServlet("\...")`
2. 定义类，继承 `HttpServlet` ，实现 `doPost` 方法
3. 再调用 DAO、DAOImpl 更改数据库

### 继承关系

Servlet 接口

- void init（config） 初始化方法

- void service(request,response) 服务方法 （收到请求自动调用）
- void destory() 销毁方法

Servlet 父类、抽象类

- genericServlet 抽象类，用于处理子类没有实现的方法（抛错误）
  - httpServlet 实现类，具体处理逻辑

### 生命周期

默认情况下：

- 第一次接收请求时，这个servlet会进行实例化（调用构造方法）、初始化（调用`init())`）、然后服务（调用`service()`）
  - 默认只会有一个 Servlet 示例
  - 单例的，线程不安全的 --> 尽量不在 Servlet 中定义、修改成员变量
- 第二次请求开始，每一次都是服务

- 当容器关闭时，其中的所有的 servlet 实例会被销毁，调用销毁方法

### Http 协议

超文本传输协议，是无状态的

请求内容

- 请求行
  - 方式、URL、HTTP版本
- 请求头
  - Host、Referer、Cookie……
- 请求体

响应内容

- 响应行
  - 协议、状态码（200）、响应状态（ok）
- 响应消息头
  - Server、Content-type……
- 响应体

### Session

会话跟踪技术，解决 http 的无状态问题。

`request.getSession`，获取、自动分配 Session

- `session.setAttribute`，保存 session 作用域

### 服务器端内部转发和客户端重定向

内部转发：同一请求、不同服务组件之间转发处理

- getdepatcher...

重定向：返回客户端一个新的服务，使其重新请求

- redirect...

### Thymeleaf 视图模板技术

辅助渲染从 DAO 获取的数据到前端

1. 引入模板
2. 将查询的数据动态显示

`th: if... unless... each... text...`

### 保存作用域

- request：一次请求有效
- session：一次会话范围有效
- application：一次应用程序范围内有效（上下文）

### 小项目

编辑、修改特定信息

- 变量的获取 `url/(fid=${name.fid})`
- 跳转到编辑页面，获取数据

## MVC Servlet优化

### Servlet 整合

问题：

- Servlet 组件过多

优化：

- 将各种 Servlet 整合到一个 `Controller` 中，作为方法
- 用路径区分操作要求
- 通过 `Switch ... case ...` 进行跳转，执行 DAO
  - 再用反射优化，`this.getClass().getDeclaredMethods();`

### DispatcherServlet 中央控制器

##### 抽取路径、反射代码

拦截、处理指定请求，修改路径再传输到 Controller

将每个 Controller 的反射代码再抽取到父类

> 注意，这样的 Controller 继承 DispatcherServlet，不能再自动调用 Init（），需要另外处理

- 解析 xml 配置文件
- 定位指定 Controller
- 读取 bean 配置，整合为 `map<id,object>`，寻找指定方法

##### 抽取重定向

Controller处理后，return 一个字符串，再交给 DispatcherServlet 处理

##### 抽取传入参数

获取参数的过程同一抽取到 DispatcherServlet

- jdk 8 新特性，通过反射获取方法参数的方法名
- 将得到的参数拆包、赋值、类型转换后，再传递给 Controller

### 初始化

重写 init（）方法，通过注解或 xml，向初始化方法中添加参数

### Service

- model：模型层
  - pojo/vo、DAO（数据访问对象，单精度方法）、BO（业务对象）
  - 在 Controller 与 DAO 间添加 Service 层
- controller：控制
- view：视图

### IOC 控制反转

实现 Bean 的自动装配，整理 xml 映射文件

- 将解析出的实例创建并存放在 beanmap 中，beanmap 存放在 BeanFactory 中，改变示例生命周期（存放到 IOC 容器中），需要时取用即可

```Java
Field propertyField = beanClazz.getDeclaredField(propertyName);
propertyField.setAccessible(true);
propertyField.set(beanobj,refobj);
```

### Filter 过滤器

拦截请求、响应

- 实现 Filter 接口，`@WebFilter("...")`
- 改写 `doFilter`，后放行（执行 Service）`filterchain.doFilter(...)`

- 接收到 Service 后，继续执行完 doFilter

过滤器链

- 执行顺序：根据全类名、xml 配置的顺序

#### 事务管理

一个 Service 需要作为一个整体，操作多个数据库时要保证同时成功或失败

- 将 try …… catch 放到 Filter 当中处理，统一回滚
- 用 `ThreadLocal` 来保证对象、线程（Connection）的同一性

##### OpenSessionInViewFilter 的实现

 事务管理封装为 TransactionManager，实现开启、提交、回滚事务的方法

- 本来分开的 commit、rollback 等，手动进行管理
- 注意内部不能 catch 异常，需要都交给 Filter 处理
  - 或者 catch 为新的异常抛出

##### ThreadLocal 的实现源码

```Java
public void set(T value)
    Thread t=Thread.currentThread();//获取当前的线程
    ThreadLocalMap map = getMap(t);//每一个线程都维护各自的一个容器(ThreadLocalMap)
    iff (map != null)
    map.set((this)value);//用map，支持多个对象存储
    else
    createMap(t, value) ;
}
```

### Listener 监听器

监听各种对象的创建、销毁；保存作用域的变化；对象在 Session 中的创建与移除、序列化与反序列化

- 将 IOC 整合到 Listener 中（监听上下文启动），提前初始化，提高响应速度（减慢启动速度）

## QQZone 笔记

### 数据库

#### 设计

先写出功能，再分析：

1. 抽取实体：用户登录信息、用户详情信息、日志、回贴、主人回复
2. 分析其中属性
   1. 用户登录信息：账号、密码、头像、昵称
   2. 用户详情信息：真实姓名、星座、血型、邮箱、手机号
   3. 日志：标题、内容、日期、作者
   4. 回复：内容、日期、作者、日志
3. 分析实体之间关系
   - 一对一 or 一对多 or 多对多

#### 范式

第一范式：列不可再分（空间尽量小）

第二范式：一张表只表达一层含义

第三范式：表中每一列和逐渐都是**直接**依赖关系（与多表连接查询权衡）

- 根据数据库查询频次、量，调整规范性

主键尽量不与业务产生联系

- 直接用自增组件

### 实体类 - POJO

定义对应属性，有关系的要对应

```java
private UserDetail userDetail//1:1
private List<Topic> topiqList//1:N
```

### 数据层 - DAO

接口、impl，实现查询

### 业务层 - Service

接口、impl，整合 DAO 层