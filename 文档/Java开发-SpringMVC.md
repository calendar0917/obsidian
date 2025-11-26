---
title: "SpringMVC"
subtitle: "SpringMVC"
description: "记录SpringMVC的学习"
date: 2025-10-12
lastmod: 2025-10-12
draft: false
toc:
 enable: true
weight: false
categories: ["开发"]
tags: ["技术"]
---

## Spring MVC

主要覆盖表述层（Controller）

- 简化接收前端参数、响应前端数据

### 工作流程

![image-20251012085938669](https://raw.githubusercontent.com/calendar0917/images/master/image-20251012085938669.png)

### 接收数据

#### 路径设置

`RequestMapping("path")` 注解

- `GetMapping("...")`

- ......

#### 接收参数

##### param

1. 参数名和 param 名相同，直接接收
2. 注解接收 `RequestParam(value = "...",required = "...")`

3. 多字符串，直接用集合接收
   - 必须要注解

4. 封装为对象接收
   - 属性名必须等于参数名

##### 路径传参

```java
@RequestMapping("{account}/{password}")
public String login(@PathVariable String account,@PathVariable String password){
	return null;
}
```

##### Json 参数

1. 定义接收的实体类
2. 用 `RequestBody` 接收

原生 Java 不支持接收 Json

- handlerMapper 中配置 Json 转换器，`EnableWebMVC`
  - 用于给`RequestMappingHandLerMapping、RequestMappingHandLerAdapter` 添加Json处理器

##### Cookie

`@CookieValue`

存 Cookie：

```java
public String save(HttpServletResponse response){
    Cookie cookie = new Cookie(name:"cookiellame", value:"root");
    response.addCookie(cookie);
    return "ok";
}
```

##### 请求头

`@RequestHeader`

##### 原生 API 对象

![image-20251012093432725](https://raw.githubusercontent.com/calendar0917/images/master/image-20251012093432725.png)

##### 共享域对象

Session、ServletContext

- 原生获取

SpringMVC 提供

- `model modeLMap map modeLAndView`

### 响应数据

#### 前后端不分离

不用 `ResponseBody`，配置 Jsp 模板地址，返回

##### 转发

`return "forward:/jsp/..."`

### 返回 Json

直接 `return User`，会被自动封装

- 需要 `@ResponseBody` 注解，不会走视图转换器
  - `ResponseBody + Controller = RestController`

### 返回静态资源

需要配置 Config

```java
//开启静态资源查找
//dispatcherServLet->handLerMapping找有没有对应的handLer->【没有->找有没有静态资源】

@Override
public void configurelDefaultServletHandling(DefaultServletHandlerConfigurer configurer）{
    configurer.enable();
}
```

## Restful 风格

规定路径设计方式、参数传递格式、选择请求方式

### 风格特点

1. 每一个URI代表1种资源；

2. 客户端使用GET、POST、PUT、DELETE4个表示操作方式的动词对服务端资源进行操作：GET用来获取资源，POST用来新建资源（也可以用于更新资源），PUT用来更新资
   源，DELETE用来删除资源；

3. 资源的表现形式是XML或者JSON；

4. 客户端与服务端之间的交互在请求之间是无状态的，从客户端到服务端的每个请求都必须包含理解请求所必需的信息。

- 用请求方式 + URI 来表示操作、对象
  - 路径传参：对应单一资源，如 id

## 其他扩展

### 全局异常处理

异常处理分类

- 编程式：单独细化处理
- 声明式：统一处理

```java
//全局异常发生，会走此类写的handLer！
//@ControLLerAdvice//可以返回逻辑视图转发和重定向的！
no usages
@RestControllerAdvice//@ResponseBody直接返回json字符串
public class GlobalExceptionHandler{
    //发生异常->ControLLerAdvice注解的类型->@ExceptionHandLer（指定的异常）->handLer
    no usages
    @ExceptionHandler(ArithmeticException.class)
    public ObjectArithmeticExceptionHandler(ArithmeticException e){
    //自定义处理异常即可handLer
    }
```

### 拦截器

`HandlerIntercepter`

拦截器 Springmvc VS 过滤器 javaWeb:
- 相似点
  - 拦截: 必须先把请求拦住，才能执行后续操作
  - 过滤: 拦截器或过滤器存在的意义就是对请求进行统一处理
  - 放行: 对请求执行了必要操作后，放请求过去，让它访问原本想要访问的资源
- 不同点
  - 工作平台不同
    - 过滤器工作在 Servlet 容器中
    - 拦截器工作在 SpringMVC 的基础上
  - 拦截的范围
    - 过滤器: 能够拦截到的最大范围是整个 Web 应用
    - 拦截器: 能够拦截到的最大范围是整个 SpringMVC 负责的请求
  - IOC 容器支持
    - 过滤器: 想得到 IOC 容器需要调用专门的工具方法，是间接的
    - 拦截器: 它自己就在 IOC 容器中，所以可以直接从 IOC 容器中装配组件，也就是可以直接得到 IOC 容器的支持

#### 使用

1. 实现 `HandlerInterceptor` 接口

2. 修改类配置拦截器 `SpringMvcConfig impLements WebMvcConfigurer`

```java
// 配置拦截
publicvoid addInterceptors(InterceptorRegistry registry){
    //配置方案1：拦截全部请求
    registry.addInterceptor(new MyInterceptor());
    
    //配置方案2：指定地址拦截.addPathPatterns（"/user/data");；
    //*任意一层字符串**任意多层字符串
    registry.addInterceptor(new MyInterceptor())
    .addPathPatterns("/user/**");
    
    //配置方案3：排除拦截排除的地址应该在拦截地址内部！
    registry.addInterceptor(new MyInterceptor())
    .addPathPatterns("/user/**").excludePathPatterns("/user/data1");
}
```

### 参数校验

hybernate 框架实现

非空校验 `@Validate`

- NotNull：包装类型不为 null
- NotEmpty：集合类型长度大于 0
- NotBlank：字符串不为 null，且不为 ""

通过 `BindingResult` 绑定错误，不直接返回