---
title: "苍穹外卖"
subtitle: "苍穹外卖"
summary: "跟做笔记，体验下开发流程"
description: "跟做笔记，体验下开发流程"
image: ""
date: 2025-10-23
lastmod: 2025-10-23
draft: false
toc:
 enable: true
hiddenFromHomePage: false
weight: false
categories: ["开发"]
tags: ["开发"]
---

## 整体架构

> 参考：[苍穹外卖项目(黑马)学习笔记DAY1_黑马外卖项目-CSDN博客](https://blog.csdn.net/wwwwz9/article/details/132228618)、[苍穹外卖——项目搭建_csdn管理端-CSDN博客](https://zengyihong.blog.csdn.net/article/details/137387552?fromshare=blogdetail&sharetype=blogdetail&sharerId=137387552&sharerefer=PC&sharesource=clmm_&sharefrom=from_link)

- 企业软件开发流程
  - 需求分析（需求规格说明说、产品原型）
  - 设计（UI设计、数据库设计、接口设计）
  - 编码（项目代码、单元测试）
  - 测试
  - 上线运维

- 角色分工
  -  项目经理（对项目整体规划安排）
  - 产品经理（需求分析）
  - UI设计师
  - 架构师
  - 开发工程师
  - 测试工程师
  - 运维工程师

### NGINX 配置

利用反向代理，实现负载均衡，隐藏内部服务器达到后端安全，以及提高访问速度。

配置信息

- global：会影响 Nginx 服务器的整体行为比如运行 Nginx 的用户和用户组
- events：配置 Nginx 的事件处理方式，包括连接数限制、事件模型等
- http: 配置文件的核心，用于配置 HTTP 服务器的行为。

```config
http {
    include       mime.types;
    default_type  application/octet-stream;
 
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';
 
    #access_log  logs/access.log  main;
 
    sendfile        on;
    #tcp_nopush     on;
 
    # 设置客户端和 Nginx 服务器之间的 Keep-Alive 连接的超时时间
    keepalive_timeout  65;
 
    #gzip  on;
	
	map $http_upgrade $connection_upgrade{
		default upgrade;
		'' close;
	}
 
	upstream webservers{
	  server 127.0.0.1:8080 weight=90 ;
	  #server 127.0.0.1:8088 weight=10 ;
	}
    # 每个 server 块代表一个特定的域名或端口的配置。
    server {
        listen       80;
        server_name  localhost;
 
        #charset koi8-r;
 
        #access_log  logs/host.access.log  main;
 
        location / {
            root   html/sky;
            index  index.html index.htm;
        }
 
        #error_page  404              /404.html;
 
        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
 
        # 反向代理,处理管理端发送的请求
        location /api/ {
			proxy_pass   http://localhost:8080/admin/;
            #proxy_pass   http://webservers/admin/;
        }
		
		# 反向代理,处理用户端发送的请求
        location /user/ {
        	# 可以设置proxy_pass、proxy_set_header、proxy_redirect
            proxy_pass   http://webservers/user/;
        }
		
		# WebSocket
		location /ws/ {
            proxy_pass   http://webservers/ws/;
			proxy_http_version 1.1;
			proxy_read_timeout 3600s;
			proxy_set_header Upgrade $http_upgrade;
			proxy_set_header Connection "$connection_upgrade";
        }
    }
```

启动前创建启动目录：

```powershell
mkdir -p temp/client_body_temp
mkdir -p temp/fastcgi_temp
mkdir -p temp/proxy_temp
mkdir -p temp/scgi_temp
mkdir -p temp/uwsgi_temp
```

### 结构分析

- sky-takeout

  - sky-common

  ![image-20251023100221220](https://raw.githubusercontent.com/calendar0917/images/master/image-20251023100221220.png)

  - sky-pojo

  ![image-20251023100235140](https://raw.githubusercontent.com/calendar0917/images/master/image-20251023100235140.png)

  - sky-server
    - 配置文件、配置类、拦截器、controller、service、mapper、启动类等

### 初始化

本地、远程仓库配置

数据库配置（导入数据库设计文档）

```shell
docker run -d \
  --name mysql-container \  # 容器名称（自定义）
  -p 3306:3306 \           # 端口映射（主机端口:容器端口）
  -e MYSQL_ROOT_PASSWORD=1234 \  # root 用户密码（必填）
  # -e MYSQL_DATABASE=your_db \             # 初始化时创建的数据库（可选）
  # -e MYSQL_USER=root \               # 初始化时创建的用户（可选）
  # -e MYSQL_PASSWORD=user_password \       # 上述用户的密码（可选）
  -v /root/docker/mysql:/var/lib/mysql \       # 数据持久化（主机目录:容器内数据目录，已补充容器内路径）
  --restart=always \                      # 容器退出时自动重启（可选）
  mysql:5.7                            # MySQL 镜像及版本
```

### 登录功能

数据库中的密码应该是**加密**后的形式，防止数据库泄露后用户密码的暴露。

### 开发工具

- APIFox
  - 帮助看文档
- Swagger-knief4j
  - 调试工具

> apifox 是设计阶段使用的工具，管理和维护接口
> Swagger 在开发阶段使用的框架，帮助后端开发人员做后端的接口测试

引入依赖：

```xml
<dependency>
  <groupId>com.github.xiaoymin</groupId>
  <artifactId>knife4j-spring-boot-starter</artifactId>
</dependency>
```

配置类：WebMvcConfiguration.java

```java
/**
      * 通过knife4j生成接口文档
      * @return
 */
@Bean
public Docket docket() {
    ApiInfo apiInfo = new ApiInfoBuilder()
    .title("苍穹外卖项目接口文档")
    .version("2.0")
    .description("苍穹外卖项目接口文档")
    .build();
    Docket docket = new Docket(DocumentationType.SWAGGER_2)
    .apiInfo(apiInfo)
    .select()
    .apis(RequestHandlerSelectors.basePackage("com.sky.controller"))
    .paths(PathSelectors.any())
    .build();
    return docket;
}
/**
      * 设置静态资源映射
      * @param registry
 */
protected void addResourceHandlers(ResourceHandlerRegistry registry) {
    registry.addResourceHandler("/doc.html").addResourceLocations("classpath:/META-INF/resources/");
    registry.addResourceHandler("/webjars/**").addResourceLocations("classpath:/META-INF/resources/webjars/");
}

```

使用说明：

![image-20251023111657732](https://raw.githubusercontent.com/calendar0917/images/master/image-20251023111657732.png)

## 员工、分类管理

- 产品原型
  - 比较直观，便于理解业务。
  - 即业务实现的基础页面样式、传递的参数
- 设计接口
- 返回数据
- 表设计

看一下Result类怎么定义：

```java
package com.sky.result;
import lombok.Data;
import java.io.Serializable;
/**
 * 后端统一返回结果
 * @param <T>
 */
@Data
public class Result<T> implements Serializable {
    private Integer code; //编码：1成功，0和其它数字为失败
    private String msg; //错误信息
    private T data; //数据
    public static <T> Result<T> success() {
        Result<T> result = new Result<T>();
        result.code = 1;
        return result;
    }
    public static <T> Result<T> success(T object) {
        Result<T> result = new Result<T>();
        result.data = object;
        result.code = 1;
        return result;
    }
    public static <T> Result<T> error(String msg) {
        Result result = new Result();
        result.msg = msg;
        result.code = 0;
        return result;
    }
}
```

看一下 Constant 类怎么定义

```java
package com.sky.constant;

/**
 * 公共字段自动填充相关常量
 */
public class AutoFillConstant {
    /**
     * 实体类中的方法名称
     */
    public static final String SET_CREATE_TIME = "setCreateTime";
    public static final String SET_UPDATE_TIME = "setUpdateTime";
    public static final String SET_CREATE_USER = "setCreateUser";
    public static final String SET_UPDATE_USER = "setUpdateUser";
}
```

### 员工添加流程

- Controller 层中创建方法

```java
    @PostMapping
    @ApiOperation("新增员工")
    public Result save(@RequestBody EmployeeDTO employeeDTO){
        log.info("新增员工：{}",employeeDTO);
        employeeService.save(employeeDTO);//该方法后续步骤会定义
        return Result.success();
    }
```

- Service 层中声明方法

```java
void save(EmployeeDTO employeeDTO);
```

- Impl 具体实现

```java
    public void save(EmployeeDTO employeeDTO) {
        Employee employee = new Employee();

        //对象属性拷贝
        BeanUtils.copyProperties(employeeDTO, employee);

        //设置账号的状态，默认正常状态 1表示正常 0表示锁定
        employee.setStatus(StatusConstant.ENABLE);

        //设置密码，默认密码123456
        employee.setPassword(DigestUtils.md5DigestAsHex(PasswordConstant.DEFAULT_PASSWORD.getBytes()));

        //设置当前记录的创建时间和修改时间
        employee.setCreateTime(LocalDateTime.now());
        employee.setUpdateTime(LocalDateTime.now());

        //设置当前记录创建人id和修改人id
        employee.setCreateUser(10L);//目前写个假数据，后期修改
        employee.setUpdateUser(10L);

        employeeMapper.insert(employee);//后续步骤定义
    }
```

- Mapper 中实现插入

```java
@Insert("insert into employee (name, username, password, phone, sex, id_number, create_time, update_time, create_user, update_user,status) " +
            "values " +
            "(#{name},#{username},#{password},#{phone},#{sex},#{idNumber},#{createTime},#{updateTime},#{createUser},#{updateUser},#{status})")
    void insert(Employee employee);
```

> 在application.yml中已开启驼峰命名，故id_number和idNumber可对应。
>
> ```yaml
> mybatis:
>   configuration:
>     #开启驼峰命名
>     map-underscore-to-camel-case: true
> ```
>

#### 完善

- 录入用户名冲突时，抛出的异常没处理

  - 通过添加全局处理器

  - ```java
    // sky-server/com.sky.hander/GlobalExceptionHander.java	
    public class GlobalExceptionHandler{
        ......
    /**
         * 处理SQL异常
         * @param ex
         * @return
         */
        @ExceptionHandler
        public Result exceptionHandler(SQLIntegrityConstraintViolationException ex){
            //Duplicate entry 'zhangsan' for key 'employee.idx_username'
            String message = ex.getMessage();
            if(message.contains("Duplicate entry")){
                String[] split = message.split(" ");
                String username = split[2];
                String msg = username + MessageConstant.ALREADY_EXISTS;
                return Result.error(msg);
            }else{
                return Result.error(MessageConstant.UNKNOWN_ERROR);
            }
        }
    }
    ```

- 新增员工时，创建、修改人 id 为固定值

  - 需要动态获取当前用户信息 --> jwt 拦截获取 id ，存入上下文

  - 拦截器的书写：

```java
@Component
@Slf4j
public class JwtTokenAdminInterceptor implements HandlerInterceptor {
    @Autowired
    private JwtProperties jwtProperties;
    /**
     * 校验jwt
     *
     * @param request
     * @param response
     * @param handler
     * @return
     * @throws Exception
     */
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //判断当前拦截到的是Controller的方法还是其他资源
        //识别到 Mapping...，会自动被包装为 HandlerMethod
        if (!(handler instanceof HandlerMethod)) {
            //当前拦截到的不是动态方法，直接放行
            return true;
        }
        //1、从请求头中获取令牌
        String token = request.getHeader(jwtProperties.getAdminTokenName());
        //2、校验令牌
        try {
            log.info("jwt校验:{}", token);
            Claims claims = JwtUtil.parseJWT(jwtProperties.getAdminSecretKey(), token);
            Long empId = Long.valueOf(claims.get(JwtClaimsConstant.EMP_ID).toString());
            log.info("当前员工id：", empId);
            //3、通过，放行
            return true;
        } catch (Exception ex) {
            //4、不通过，响应401状态码
            response.setStatus(401);
            return false;
        }
    }
}
```

ThreadLocal：

- 并不是一个Thread，而是Thread的局部变量。
- 为每个线程提供单独一份存储空间，具有线程隔离的效果，只有在线程内才能获取到对应的值，线程外则不能访问。

> **常用方法：**
>
> - public void set(T value) 	设置当前线程的线程局部变量的值
> - public T get() 		返回当前线程所对应的线程局部变量的值
> - public void remove()        移除当前线程的线程局部变量

看一下封装了的 ThreadLocal 工具类

```java
// 在sky-common/com.sky.context
    public class BaseContext {
    public static ThreadLocal<Long> threadLocal = new ThreadLocal<>();
    public static void setCurrentId(Long id) {
        threadLocal.set(id);
    }
    public static Long getCurrentId() {
        return threadLocal.get();
    }
    public static void removeCurrentId() {
        threadLocal.remove();
    }
}
```

### 分页查询

- 封装 EmployeePageQueryDTO 来接收参数

- 封装 PageResult 对象，来返回分页参数

  - 包含总记录数、当前页数据集合

  - ```java
    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    public class PageResult implements Serializable {
        private long total; //总记录数
        private List records; //当前页数据集合
    }
    ```

  - 后续用 `Result<PageResult>` 返回

- controller 层

```java
	@GetMapping
    @ApiOperation(value = "分页查询")
    public Result<PageResult> pageQuery(@RequestParam EmployeePageQueryDTO employeePageQueryDTO){
        log.info("分页查询{}",employeePageQueryDTO);
        PageResult pageResult = employeeService.pageQuery(employeePageQueryDTO);
        return Result.success(pageResult);
    }
```

- Service 层

  - 关注 PageHelper 的使用

  - > 来自：
    >
    > ```
    > <dependency>
    >    <groupId>com.github.pagehelper</groupId>
    >    <artifactId>pagehelper-spring-boot-starter</artifactId>
    >    <version>${pagehelper}</version>
    > </dependency>
    > ```

```java
public PageResult pageQuery(EmployeePageQueryDTO employeePageQueryDTO) {
    // SQL 语法：select * from employee limit 0,10
    // 传入 startPage，然后查询，用 Page<T> 接收
        PageHelper.startPage(employeePageQueryDTO.getPage(),employeePageQueryDTO.getPageSize());
        Page<Employee> page = employeeMapper.pageQuery(employeePageQueryDTO);
		// 从封装好的 page 对象中取出结果
        long total = page.getTotal();
        List<Employee> records = page.getResult();
        return new PageResult(total,records);
    }
```

- Mapper
  - 复杂查询用 .xml

```xml
<mapper namespace="com.sky.mapper.EmployeeMapper">
    <select id="pageQuery" resultType="com.sky.entity.Employee">
        select * from employee
        <where>
            <if test="name != null and name != ''">
                and name like concat('%',#{name},'%')
            </if>
        </where>
        order by create_time desc
    </select>
</mapper>
```

#### 完善

- 日期显示格式有问题
  - 方法一：在属性上加注解`@JsonFormat(patter = "yyyy-MM-dd HH:mm:ss")`
  - 方法二：在WebMvcConfiguration中扩展SpringMVC的**消息转换器**，统一对日期类型进行格式处理

消息转换器：

- 当后端向前端返回 ResponseBody，或接收前端 RequestBody 时，会自动调用

```java
	/**
     * 扩展Spring MVC框架的消息转化器
     * @param converters
     */
    protected void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        log.info("扩展消息转换器...");
        //创建一个消息转换器对象
        MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter();
        //需要为消息转换器设置一个对象转换器，对象转换器可以将Java对象序列化为json数据
        converter.setObjectMapper(new JacksonObjectMapper());
        //将自己的消息转化器加入容器中
        converters.add(0,converter);
    }
```

Json 对象映射器

```java
package com.sky.json;

import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.module.SimpleModule;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateDeserializer;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalDateTimeDeserializer;
import com.fasterxml.jackson.datatype.jsr310.deser.LocalTimeDeserializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateSerializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalDateTimeSerializer;
import com.fasterxml.jackson.datatype.jsr310.ser.LocalTimeSerializer;

import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;

import static com.fasterxml.jackson.databind.DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES;

/**
 * 对象映射器:基于jackson将Java对象转为json，或者将json转为Java对象
 * 将JSON解析为Java对象的过程称为 [从JSON反序列化Java对象]
 * 从Java对象生成JSON的过程称为 [序列化Java对象到JSON]
 */
public class JacksonObjectMapper extends ObjectMapper {

    public static final String DEFAULT_DATE_FORMAT = "yyyy-MM-dd";
    //public static final String DEFAULT_DATE_TIME_FORMAT = "yyyy-MM-dd HH:mm:ss";
    public static final String DEFAULT_DATE_TIME_FORMAT = "yyyy-MM-dd HH:mm";
    public static final String DEFAULT_TIME_FORMAT = "HH:mm:ss";

    public JacksonObjectMapper() {
        super();
        //收到未知属性时不报异常
        this.configure(FAIL_ON_UNKNOWN_PROPERTIES, false);

        //反序列化时，属性不存在的兼容处理
        this.getDeserializationConfig().withoutFeatures(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);

        SimpleModule simpleModule = new SimpleModule()
                .addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)))
                .addDeserializer(LocalDate.class, new LocalDateDeserializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)))
                .addDeserializer(LocalTime.class, new LocalTimeDeserializer(DateTimeFormatter.ofPattern(DEFAULT_TIME_FORMAT)))
                .addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)))
                .addSerializer(LocalDate.class, new LocalDateSerializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)))
                .addSerializer(LocalTime.class, new LocalTimeSerializer(DateTimeFormatter.ofPattern(DEFAULT_TIME_FORMAT)));

        //注册功能模块 例如，可以添加自定义序列化器和反序列化器
        this.registerModule(simpleModule);
    }
}
```

### 启用、禁用

数据层处理：直接用 update 来更新！整合到一起

- Controller
  - 注意接收路径参数、query 参数

```java
    @PostMapping("/status/{status}")
    @ApiOperation("启用禁用员工账号")
    public Result startOrStop(@PathVariable Integer status,Long id){
        log.info("启用禁用员工账号：{},{}",status,id);
        employeeService.startOrStop(status,id);//后绪步骤定义
        return Result.success();
    }
```

- Service

```java
    public void startOrStop(Integer status, Long id) {
        Employee employee = Employee.builder()
                .status(status)
                .id(id)
                .build();

        employeeMapper.update(employee);
    }
```

- Mapper .xml

```xml
<update id="update" parameterType="Employee">
        update employee
        <set>
            <if test="name != null">name = #{name},</if>
            <if test="username != null">username = #{username},</if>
            <if test="password != null">password = #{password},</if>
            <if test="phone != null">phone = #{phone},</if>
            <if test="sex != null">sex = #{sex},</if>
            <if test="idNumber != null">id_Number = #{idNumber},</if>
            <if test="updateTime != null">update_Time = #{updateTime},</if>
            <if test="updateUser != null">update_User = #{updateUser},</if>
            <if test="status != null">status = #{status},</if>
        </set>
        where id = #{id}
    </update>
```

### 编辑员工

#### 根据 id 查询员工

- Controller

```java
    @GetMapping("/{id}")
    @ApiOperation("根据id查询员工信息")
    public Result<Employee> getById(@PathVariable Long id){
        Employee employee = employeeService.getById(id);
        return Result.success(employee);
    }
```

- Service
  - 注意：密码回显前要隐藏！

```java
public Employee getById(Long id) {
        Employee employee = employeeMapper.getById(id);
        employee.setPassword("****");
        return employee;
    }
```

- Mapper

```java
@Select("select * from employee where id = #{id}")
Employee getById(Long id);
```

#### 修改员工信息

直接用 update 将原来的覆盖即可

- Controller
  - 注意方式为 Put！

```java
@PutMapping
@ApiOperation("编辑员工信息")
public Result update(@RequestBody EmployeeDTO employeeDTO){
        log.info("编辑员工信息：{}", employeeDTO);
        employeeService.update(employeeDTO);
        return Result.success();
    }
```

- Service

```java
public void update(EmployeeDTO employeeDTO) {
        Employee employee = new Employee();
        BeanUtils.copyProperties(employeeDTO, employee);

        employee.setUpdateTime(LocalDateTime.now());
        employee.setUpdateUser(BaseContext.getCurrentId());

        employeeMapper.update(employee);
    }
```

- Mapper
  - 就是上面的 update

## 公共字段填充

抽取需要重复赋值的部分，如 `setCreateTime(...)` 等

使用 **AOP 编程**！

**实现步骤：**

1. 自定义注解 AutoFill，用于标识需要进行公共字段自动填充的方法

2. 自定义切面类 AutoFillAspect，统一拦截加入了 AutoFill 注解的方法，通过反射为公共字段赋值

3. 在 Mapper 的方法上加入 AutoFill 注解

- 自定义注解

```java
// sky-server/com.sky.annotation
package com.sky.annotation;
import ...

/**
 * 自定义注解，用于标识某个方法需要进行功能字段自动填充处理
 */
@Target(ElementType.METHOD)  // 定义声明周期
@Retention(RetentionPolicy.RUNTIME) // 定义使用范围
public @interface AutoFill {
    //数据库操作类型：UPDATE INSERT
    OperationType value();
}
```

> 其中，OperationType 在 sky-common 中定义：
>
> ```java
> package com.sky.enumeration;
> /**
>  * 数据库操作类型
>  */
> public enum OperationType {
>     UPDATE,
>     INSERT
> }
> ```

- 自定义切面

```java
public void autoFill(JoinPoint joinPoint){
        log.info("开始进行公共字段自动填充...");

        //获取到当前被拦截的方法上的数据库操作类型
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();//方法签名对象
        AutoFill autoFill = signature.getMethod().getAnnotation(AutoFill.class);//获得方法上的注解对象
        OperationType operationType = autoFill.value();//获得数据库操作类型

        //获取到当前被拦截的方法的参数--实体对象
        Object[] args = joinPoint.getArgs();
        if(args == null || args.length == 0){
            return;
        }

        Object entity = args[0];

        //准备赋值的数据
        LocalDateTime now = LocalDateTime.now();
        Long currentId = BaseContext.getCurrentId();

        //根据当前不同的操作类型，为对应的属性通过反射来赋值
        if(operationType == OperationType.INSERT){
            //为4个公共字段赋值
            try {
                Method setCreateTime = entity.getClass().getDeclaredMethod(AutoFillConstant.SET_CREATE_TIME, LocalDateTime.class);
                Method setCreateUser = entity.getClass().getDeclaredMethod(AutoFillConstant.SET_CREATE_USER, Long.class);
                Method setUpdateTime = entity.getClass().getDeclaredMethod(AutoFillConstant.SET_UPDATE_TIME, LocalDateTime.class);
                Method setUpdateUser = entity.getClass().getDeclaredMethod(AutoFillConstant.SET_UPDATE_USER, Long.class);

                //通过反射为对象属性赋值
                setCreateTime.invoke(entity,now);
                setCreateUser.invoke(entity,currentId);
                setUpdateTime.invoke(entity,now);
                setUpdateUser.invoke(entity,currentId);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }else if(operationType == OperationType.UPDATE){
            //为2个公共字段赋值
            try {
                Method setUpdateTime = entity.getClass().getDeclaredMethod(AutoFillConstant.SET_UPDATE_TIME, LocalDateTime.class);
                Method setUpdateUser = entity.getClass().getDeclaredMethod(AutoFillConstant.SET_UPDATE_USER, Long.class);

                //通过反射为对象属性赋值
                setUpdateTime.invoke(entity,now);
                setUpdateUser.invoke(entity,currentId);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
```

## 菜品管理

### 文件上传的配置

实现图片的上传、多表操作

- 文件上传
  - 使用阿里云的 OSS

配置 application-dev.yml

```yml
sky:
  alioss:
    endpoint: oss-cn-hangzhou.aliyuncs.com
    access-key-id: ...
    access-key-secret: ...
    bucket-name: sky-takeout-calendar
```

application.yml

```yml
spring:
  profiles:
    active: dev    #设置环境
sky:
  alioss:
    endpoint: ${sky.alioss.endpoint}
    access-key-id: ${sky.alioss.access-key-id}
    access-key-secret: ${sky.alioss.access-key-secret}
    bucket-name: ${sky.alioss.bucket-name}
```

读取配置

```java
// sky-commom/com.sky.properties
package com.sky.properties;

@Component
@ConfigurationProperties(prefix = "sky.alioss")
@Data
public class AliOssProperties {
    private String endpoint;
    private String accessKeyId;
    private String accessKeySecret;
    private String bucketName;
}
```

配置类，用于生成 OSS 工具类对象

```java
// sky-server/com.sky.config
package com.sky.config;
/**
 * 配置类，用于创建AliOssUtil对象
 */
@Configuration
@Slf4j
public class OssConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public AliOssUtil aliOssUtil(AliOssProperties aliOssProperties){
        log.info("开始创建阿里云文件上传工具类对象：{}",aliOssProperties);
        return new AliOssUtil(aliOssProperties.getEndpoint(),
                aliOssProperties.getAccessKeyId(),
                aliOssProperties.getAccessKeySecret(),
                aliOssProperties.getBucketName());
    }
}
```

> AliOssUtil 在 sky-common 中定义
>
> ```java
> package com.sky.utils;
> @Data
> @AllArgsConstructor
> @Slf4j
> public class AliOssUtil {
> 
>     private String endpoint;
>     private String accessKeyId;
>     private String accessKeySecret;
>     private String bucketName;
> 
>     /**
>      * 文件上传
>      */
>     public String upload(byte[] bytes, String objectName) {
> 
>         // 创建OSSClient实例。
>         OSS ossClient = new OSSClientBuilder().build(endpoint, accessKeyId, accessKeySecret);
> 
>         try {
>             // 创建PutObject请求。
>             ossClient.putObject(bucketName, objectName, new ByteArrayInputStream(bytes));
>         } catch (OSSException oe) {
>             System.out.println("Caught an OSSException, which means your request made it to OSS, "
>                     + "but was rejected with an error response for some reason.");
>             System.out.println("Error Message:" + oe.getErrorMessage());
>             System.out.println("Error Code:" + oe.getErrorCode());
>             System.out.println("Request ID:" + oe.getRequestId());
>             System.out.println("Host ID:" + oe.getHostId());
>         } catch (ClientException ce) {
>             System.out.println("Caught an ClientException, which means the client encountered "
>                     + "a serious internal problem while trying to communicate with OSS, "
>                     + "such as not being able to access the network.");
>             System.out.println("Error Message:" + ce.getMessage());
>         } finally {
>             if (ossClient != null) {
>                 ossClient.shutdown();
>             }
>         }
> 
>         //文件访问路径规则 https://BucketName.Endpoint/ObjectName
>         StringBuilder stringBuilder = new StringBuilder("https://");
>         stringBuilder
>                 .append(bucketName)
>                 .append(".")
>                 .append(endpoint)
>                 .append("/")
>                 .append(objectName);
> 
>         log.info("文件上传到:{}", stringBuilder.toString());
> 
>         return stringBuilder.toString();
>     }
> }
> ```
>
> 用 putobject 上传，然后捕获异常、回显上传路径

CommonController 接口

```java
package com.sky.controller.admin;

/**
 * 通用接口
 */
@RestController
@RequestMapping("/admin/common")
@Api(tags = "通用接口")
@Slf4j
public class CommonController {

    @Autowired
    private AliOssUtil aliOssUtil;

    /**
     * 文件上传
     * @param file
     * @return
     */
    @PostMapping("/upload")
    @ApiOperation("文件上传")
    public Result<String> upload(MultipartFile file){
        log.info("文件上传：{}",file);
        try {
            //原始文件名
            String originalFilename = file.getOriginalFilename();
            //截取原始文件名的后缀   dfdfdf.png
            String extension = originalFilename.substring(originalFilename.lastIndexOf("."));
            //构造新文件名称
            String objectName = UUID.randomUUID().toString() + extension;
            //文件的请求路径
            String filePath = aliOssUtil.upload(file.getBytes(), objectName);
            return Result.success(filePath);
        } catch (IOException e) {
            log.error("文件上传失败：{}", e);
        }

        return Result.error(MessageConstant.UPLOAD_FAILED);
    }
}
```

### 新增菜品

多表操作、批量添加的处理

- Controller
  - 注意：注入的注解是 `@RequiredArgsConstructor`，且用 `private final`

```java
    @PostMapping
    @ApiOperation("新增菜品")
    public Result save(@RequestBody DishDTO dishDTO) {
        log.info("新增菜品：{}", dishDTO);
        dishService.saveWithFlavor(dishDTO);//后绪步骤开发
        return Result.success();
    }
}
```

- Service
  - 用事务解决同步性问题

```java
package com.sky.service.impl;
@Service
@Slf4j
public class DishServiceImpl implements DishService {

    @Autowired
    private DishMapper dishMapper;
    @Autowired
    private DishFlavorMapper dishFlavorMapper;

    /**
     * 新增菜品和对应的口味
     *
     * @param dishDTO
     */
    @Transactional
    public void saveWithFlavor(DishDTO dishDTO) {

        Dish dish = new Dish();
        BeanUtils.copyProperties(dishDTO, dish);

        //向菜品表插入1条数据
        dishMapper.insert(dish);//后绪步骤实现

        //获取insert语句生成的主键值
        Long dishId = dish.getId();
        
		// ！！！批量插入数据
        List<DishFlavor> flavors = dishDTO.getFlavors();
        if (flavors != null && flavors.size() > 0) {
            flavors.forEach(dishFlavor -> {
                dishFlavor.setDishId(dishId);
            });
            //向口味表插入n条数据
            dishFlavorMapper.insertBatch(flavors);//后绪步骤实现
        }
    }

}
```

- Mapper
  - 主要看 insetBatch，用 foreach 遍历

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.sky.mapper.DishFlavorMapper">
    <insert id="insertBatch">
        insert into dish_flavor (dish_id, name, value) VALUES
        <foreach collection="flavors" item="df" separator=",">
            (#{df.dishId},#{df.name},#{df.value})
        </foreach>
    </insert>
</mapper>
```

### 菜品分页查询

类似前面的分页查询，但是新增了多表查询

- Controller

```java
@GetMapping("/page")
@ApiOperation(value = "菜品分页查询")
public Result<PageResult> page(DishPageQueryDTO dishPageQueryDTO){
        log.info("菜品分页查询{}",dishPageQueryDTO);
        PageResult pageResult = dishService.page(dishPageQueryDTO);
        return Result.success(pageResult);
    }
```

- Service

```java
    @Override
    public PageResult page(DishPageQueryDTO dishPageQueryDTO) {
        PageHelper.startPage(dishPageQueryDTO.getPage(),dishPageQueryDTO.getPageSize());
        Page<DishVO> records = dishMapper.page(dishPageQueryDTO);
        Long total = records.getTotal();
        List<DishVO> dishVOS = records.getResult();
        PageResult pageResult = new PageResult();
        pageResult.setTotal(total);
        pageResult.setRecords(dishVOS);
        return pageResult;
    }
```

- Mapper
  - 注意多表查询

```xml
    <select id="pageQuery" resultType="com.sky.vo.DishVO">
        select d.* , c.name as categoryName from dish d left outer join category c on d.category_id = c.id
        <where>
            <if test="name != null">
                and d.name like concat('%',#{name},'%')
            </if>
            <if test="categoryId != null">
                and d.category_id = #{categoryId}
            </if>
            <if test="status != null">
                and d.status = #{status}
            </if>
        </where>
        order by d.create_time desc
    </select>
```

### 删除菜品

涉及较复杂的多表关联删除操作

> **业务规则：**
>
> - 可以一次删除一个菜品，也可以批量删除菜品
> - 起售中的菜品不能删除
> - 被套餐关联的菜品不能删除
> - 删除菜品后，关联的口味数据也需要删除掉

菜品的返回需要包装一个 VO 类

- Controller

```java
    @DeleteMapping
    @ApiOperation("菜品批量删除")
    public Result delete(@RequestParam List<Long> ids) {
        log.info("菜品批量删除：{}", ids);
        dishService.deleteBatch(ids);//后绪步骤实现
        return Result.success();
    }
```

- Service
  - 注意判断逻辑

```java
@Transactional//事务
    public void deleteBatch(List<Long> ids) {
        //判断当前菜品是否能够删除---是否存在起售中的菜品？？
        for (Long id : ids) {
            Dish dish = dishMapper.getById(id);//后绪步骤实现
            if (dish.getStatus() == StatusConstant.ENABLE) {
                //当前菜品处于起售中，不能删除
                throw new DeletionNotAllowedException(MessageConstant.DISH_ON_SALE);
            }
        }

        //判断当前菜品是否能够删除---是否被套餐关联了？？
        List<Long> setmealIds = setmealDishMapper.getSetmealIdsByDishIds(ids);
        if (setmealIds != null && setmealIds.size() > 0) {
            //当前菜品被套餐关联了，不能删除
            throw new DeletionNotAllowedException(MessageConstant.DISH_BE_RELATED_BY_SETMEAL);
        }

        //删除菜品表中的菜品数据
        for (Long id : ids) {
            dishMapper.deleteById(id);//后绪步骤实现
            //删除菜品关联的口味数据
            dishFlavorMapper.deleteByDishId(id);//后绪步骤实现
        }
    }
```

- Mapper
  - 注意删除多个

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.sky.mapper.SetmealDishMapper">
    <select id="getSetmealIdsByDishIds" resultType="java.lang.Long">
        select setmeal_id from setmeal_dish where dish_id in
        <foreach collection="dishIds" item="dishId" separator="," open="(" close=")">
            #{dishId}
        </foreach>
    </select>
</mapper>
```

> 异常类编写：
>
> ```java
> package com.sky.exception;
> public class DeletionNotAllowedException extends BaseException {
> 
>     public DeletionNotAllowedException(String msg) {
>         super(msg);
>     }
> }
> ```

### 修改菜品

#### 查询

需要将 flavor 也添加进去 --> 需要新的 VO 来存储

- Controller

```java
@GetMapping("/{id}")
    @ApiOperation("根据id查询菜品")
    public Result<DishVO> getById(@PathVariable Long id) {
        log.info("根据id查询菜品：{}", id);
        DishVO dishVO = dishService.getByIdWithFlavor(id);//后绪步骤实现
        return Result.success(dishVO);
    }
```

- Service

```java
    public DishVO getByIdWithFlavor(Long id) {
        //根据id查询菜品数据
        Dish dish = dishMapper.getById(id);

        //根据菜品id查询口味数据
        List<DishFlavor> dishFlavors = dishFlavorMapper.getByDishId(id);//后绪步骤实现

        //将查询到的数据封装到VO
        DishVO dishVO = new DishVO();
        BeanUtils.copyProperties(dish, dishVO);
        dishVO.setFlavors(dishFlavors);

        return dishVO;
    }
```

- Mapper

```java
    @Select("select * from dish_flavor where dish_id = #{dishId}")
    List<DishFlavor> getByDishId(Long dishId);
```

#### 修改

- Controller

```java
    @PutMapping
    @ApiOperation("修改菜品")
    public Result update(@RequestBody DishDTO dishDTO) {
        log.info("修改菜品：{}", dishDTO);
        dishService.updateWithFlavor(dishDTO);
        return Result.success();
    }
```

- Service

```java
    public void updateWithFlavor(DishDTO dishDTO) {
        Dish dish = new Dish();
        BeanUtils.copyProperties(dishDTO, dish);

        //修改菜品表基本信息
        dishMapper.update(dish);

        //删除原有的口味数据
        dishFlavorMapper.deleteByDishId(dishDTO.getId());

        //重新插入口味数据
        List<DishFlavor> flavors = dishDTO.getFlavors();
        if (flavors != null && flavors.size() > 0) {
            flavors.forEach(dishFlavor -> {
                dishFlavor.setDishId(dishDTO.getId());
            });
            //向口味表插入n条数据
            dishFlavorMapper.insertBatch(flavors);
        }
    }
```

- Mapper
  - 注意**更新**的编写，if test 标签

```xml
<update id="update">
        update dish
        <set>
            <if test="name != null">name = #{name},</if>
            <if test="categoryId != null">category_id = #{categoryId},</if>
            <if test="price != null">price = #{price},</if>
            <if test="image != null">image = #{image},</if>
            <if test="description != null">description = #{description},</if>
            <if test="status != null">status = #{status},</if>
            <if test="updateTime != null">update_time = #{updateTime},</if>
            <if test="updateUser != null">update_user = #{updateUser},</if>
        </set>
        where id = #{id}
</update>
```

## 套餐管理

### 新增套餐

- 首先需要新增根据套餐类型查询菜品，回显
  - 注意返回值！

```java
// DishController
    @GetMapping("/list")
    @ApiOperation(value = "根据分类id查询菜品")
    public Result<List<DishVO>> getByCategoryId(@RequestParam Long categoryId){
        log.info("根据分类id查询菜品{}",categoryId);
        List<DishVO> dishVOS = dishService.getByCategoryId(categoryId);
        return Result.success(dishVOS);
    }
```

- 然后正常添加 setmeal 的 insert 即可

### 套餐分页查询

类似地，多了参数，多写几个 if test 就可以了

- 注意，`Page<Setmeal> records = setmealMapper.pageQuerySetmeal(setmealPageQueryDTO);`，返回的不是 page 对象……

### 删除套餐

- sql 不会写：
  - 接收参数要用 `@RequestParam List<Long> ids`
    - 绑定参数、参数可选、数组参数

```xml
<delete id="deleteSetmeals">
    delete from setmeal where id in
    <foreach collection="ids" item="id" open="(" close=")" separator=",">
        #{id}
    </foreach>
</delete>
```

### 修改套餐

update 语句还不太会写

参数还是要用 `@RequestBody` 接收

```xml
<update id="update">
    update setmeal
    <set>
        <if test="categoryId != null">category_id = #{categoryId},</if>
        <if test="description != null and description != ''">description = #{description},</if>
        <if test="image != null and image != ''">image = #{image},</if>
        <if test="name != null and name != ''">name = #{name},</if>
        <if test="price != null">price = #{price},</if>
        <if test="status != null">status = #{status},</if>
        <if test="updateTime != null">update_time = #{updateTime},</if>
        <if test="updateUser != null">update_user = #{updateUser}</if>
    </set>
    where id = #{id}
</update>
```

### 启用、禁用套餐

没什么问题

## Redis - 店铺营业状态

docker 部署

```shell
docker run -d \
  --name redis-4.0.0 \
  -p 6379:6379 \
  -v /root/docker/redis/data:/data \
  -v /root/docker/redis/conf/redis.conf:/etc/redis/redis.conf \
  redis:4.0.0 \
  redis-server /etc/redis/redis.conf
```

### Spring Data Redis 的使用

#### 配置

- 导入maven坐标

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

- 配置 dev.yml,并导入到 application.tml 中

```xml
spring:
  profiles:
    active: dev
  redis:
    host: ${sky.redis.host}
    port: ${sky.redis.port}
    password: ${sky.redis.password}
    database: ${sky.redis.database}
```

- 配置类

```java
package com.sky.config;
@Configuration
@Slf4j
public class RedisConfiguration {

    @Bean
    public RedisTemplate redisTemplate(RedisConnectionFactory redisConnectionFactory){
        log.info("开始创建redis模板对象...");
        RedisTemplate redisTemplate = new RedisTemplate();
        //设置redis的连接工厂对象
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        //设置redis key的序列化器
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        return redisTemplate;
    }
}
```

> - Spring Boot 内置了 `RedisAutoConfiguration` 自动配置类，会根据 `application.yml` 中的 `spring.redis` 前缀配置，自动创建 `RedisConnectionFactory` 实例
>
> - 当前配置类不是必须的，因为 Spring Boot 框架会自动装配 RedisTemplate 对象，但是默认的key序列化器为 JdkSerializationRedisSerializer，导致我们存到 Redis 中后的数据和原始数据有差别，故设置为 StringRedisSerializer 序列化器。

#### 通过配置类操作 Redis

- 字符串型

```java
	/**
     * 操作字符串类型的数据
     */
    public void testString(){
        // set get setex setnx
        redisTemplate.opsForValue().set("name","小明");
        String city = (String) redisTemplate.opsForValue().get("name");
        System.out.println(city);
        redisTemplate.opsForValue().set("code","1234",3, TimeUnit.MINUTES);
        redisTemplate.opsForValue().setIfAbsent("lock","1");
        redisTemplate.opsForValue().setIfAbsent("lock","2");
    }
```

- 哈希型

```java
	/**
     * 操作哈希类型的数据
     */
    public void testHash(){
        //hset hget hdel hkeys hvals
        HashOperations hashOperations = redisTemplate.opsForHash();

        hashOperations.put("100","name","tom");
        hashOperations.put("100","age","20");

        String name = (String) hashOperations.get("100", "name");
        System.out.println(name);

        Set keys = hashOperations.keys("100");
        System.out.println(keys);

        List values = hashOperations.values("100");
        System.out.println(values);

        hashOperations.delete("100","age");
    }
```

- 列表类型

```java
/**
 * 操作列表类型的数据
 */
public void testList(){
    //lpush lrange rpop llen
    ListOperations listOperations = redisTemplate.opsForList();

    listOperations.leftPushAll("mylist","a","b","c");
    listOperations.leftPush("mylist","d");

    List mylist = listOperations.range("mylist", 0, -1);
    System.out.println(mylist);

    listOperations.rightPop("mylist");

    Long size = listOperations.size("mylist");
    System.out.println(size);
}
```

- 集合类型

```java
/**
 * 操作集合类型的数据
 */
@Test
public void testSet(){
    //sadd smembers scard sinter sunion srem
    SetOperations setOperations = redisTemplate.opsForSet();

    setOperations.add("set1","a","b","c","d");
    setOperations.add("set2","a","b","x","y");

    Set members = setOperations.members("set1");
    System.out.println(members);

    Long size = setOperations.size("set1");
    System.out.println(size);

    Set intersect = setOperations.intersect("set1", "set2");
    System.out.println(intersect);

    Set union = setOperations.union("set1", "set2");
    System.out.println(union);

    setOperations.remove("set1","a","b");
}
```

- 有序集合类型

```java
/**
 * 操作有序集合类型的数据
 */
@Test
public void testZset(){
    //zadd zrange zincrby zrem
    ZSetOperations zSetOperations = redisTemplate.opsForZSet();

    zSetOperations.add("zset1","a",10);
    zSetOperations.add("zset1","b",12);
    zSetOperations.add("zset1","c",9);

    Set zset1 = zSetOperations.range("zset1", 0, -1);
    System.out.println(zset1);

    zSetOperations.incrementScore("zset1","c",10);

    zSetOperations.remove("zset1","a","b");
}
```

- 通用命令

```java
	/**
     * 通用命令操作
     */
    @Test
    public void testCommon(){
        //keys exists type del
        Set keys = redisTemplate.keys("*");
        System.out.println(keys);

        Boolean name = redisTemplate.hasKey("name");
        Boolean set1 = redisTemplate.hasKey("set1");

        for (Object key : keys) {
            DataType type = redisTemplate.type(key);
            System.out.println(type.name());
        }

        redisTemplate.delete("mylist");
    }
```

### 店铺营业状态设置

> **接口设计：**
>
> - 设置营业状态
> - 管理端查询营业状态
> - 用户端查询营业状态

- admin - ShopController
  - user 类似，重名的 class 要指定 RestController 名称，否则自动装配会分辨不了

```java
package com.sky.controller.admin;

@RestController("adminShopController")
@RequestMapping("/admin/shop")
@Api(tags = "店铺相关接口")
@Slf4j
public class ShopController {

    public static final String KEY = "SHOP_STATUS";

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * 设置店铺的营业状态
     * @param status
     * @return
     */
    @PutMapping("/{status}")
    @ApiOperation("设置店铺的营业状态")
    public Result setStatus(@PathVariable Integer status){
        log.info("设置店铺的营业状态为：{}",status == 1 ? "营业中" : "打烊中");
        redisTemplate.opsForValue().set(KEY,status);
        return Result.success();
    }
    /**
     * 获取店铺的营业状态
     * @return
     */
    @GetMapping("/status")
    @ApiOperation("获取店铺的营业状态")
    public Result<Integer> getStatus(){
        Integer status = (Integer) redisTemplate.opsForValue().get(KEY);
        log.info("获取到店铺的营业状态为：{}",status == 1 ? "营业中" : "打烊中");
        return Result.success(status);
    }
}
```

### 用户、管理接口分离

WebConfiguration 中配置扫描

```java
	@Bean
    public Docket docket1(){
        log.info("准备生成接口文档...");
        ApiInfo apiInfo = new ApiInfoBuilder()
                .title("苍穹外卖项目接口文档")
                .version("2.0")
                .description("苍穹外卖项目接口文档")
                .build();

        Docket docket = new Docket(DocumentationType.SWAGGER_2)
                .groupName("管理端接口")
                .apiInfo(apiInfo)
                .select()
                //指定生成接口需要扫描的包
                .apis(RequestHandlerSelectors.basePackage("com.sky.controller.admin"))
                .paths(PathSelectors.any())
                .build();

        return docket;
    }

    @Bean
    public Docket docket2(){
        log.info("准备生成接口文档...");
        ApiInfo apiInfo = new ApiInfoBuilder()
                .title("苍穹外卖项目接口文档")
                .version("2.0")
                .description("苍穹外卖项目接口文档")
                .build();

        Docket docket = new Docket(DocumentationType.SWAGGER_2)
                .groupName("用户端接口")
                .apiInfo(apiInfo)
                .select()
                //指定生成接口需要扫描的包
                .apis(RequestHandlerSelectors.basePackage("com.sky.controller.user"))
                .paths(PathSelectors.any())
                .build();

        return docket;
    }
```

## HttpClient

> **HttpClient作用：**
>
> - 发送HTTP请求
> - 接收响应数据
> - 使用扫描支付、查看地图、获取验证码、查看天气等功能

- 导入坐标

```xml
<dependency>
	<groupId>org.apache.httpcomponents</groupId>
	<artifactId>httpclient</artifactId>
	<version>4.5.13</version>
</dependency>
```

- 核心 API

  - HttpClient：Http客户端对象类型，使用该类型对象可发起Http请求。

  - HttpClients：可认为是构建器，可创建HttpClient对象。

  - CloseableHttpClient：实现类，实现了HttpClient接口。

  - HttpGet：Get方式请求类型。

  - HttpPost：Post方式请求类型。

- 发送请求步骤：

  - 创建HttpClient对象
  - 创建Http请求对象
  - 调用HttpClient的execute方法发送请求


### 案例

- get

```java
@SpringBootTest
public class HttpClientTest {

    /**
     * 测试通过httpclient发送GET方式的请求
     */
    @Test
    public void testGET() throws Exception{
        //创建httpclient对象
        CloseableHttpClient httpClient = HttpClients.createDefault();

        //创建请求对象
        HttpGet httpGet = new HttpGet("http://localhost:8080/user/shop/status");

        //发送请求，接受响应结果
        CloseableHttpResponse response = httpClient.execute(httpGet);

        //获取服务端返回的状态码
        int statusCode = response.getStatusLine().getStatusCode();
        System.out.println("服务端返回的状态码为：" + statusCode);

        HttpEntity entity = response.getEntity();
        String body = EntityUtils.toString(entity);
        System.out.println("服务端返回的数据为：" + body);

        //关闭资源
        response.close();
        httpClient.close();
    }
}
```

- post

```java
	/**
     * 测试通过httpclient发送POST方式的请求
     */
    @Test
    public void testPOST() throws Exception{
        // 创建httpclient对象
        CloseableHttpClient httpClient = HttpClients.createDefault();

        //创建请求对象
        HttpPost httpPost = new HttpPost("http://localhost:8080/admin/employee/login");

        JSONObject jsonObject = new JSONObject();
        jsonObject.put("username","admin");
        jsonObject.put("password","123456");

        StringEntity entity = new StringEntity(jsonObject.toString());
        //指定请求编码方式
        entity.setContentEncoding("utf-8");
        //数据格式
        entity.setContentType("application/json");
        httpPost.setEntity(entity);

        //发送请求
        CloseableHttpResponse response = httpClient.execute(httpPost);

        //解析返回结果
        int statusCode = response.getStatusLine().getStatusCode();
        System.out.println("响应码为：" + statusCode);

        HttpEntity entity1 = response.getEntity();
        String body = EntityUtils.toString(entity1);
        System.out.println("响应数据为：" + body);

        //关闭资源
        response.close();
        httpClient.close();
    }
```

## 微信小程序开发

### 小程序目录结构

- 小程序包含一个描述整体程序的 app 和多个描述各自页面的 page。一个小程序主体部分由三个文件组成，必须放在项目的根目录，如下：
  - **app.js：**必须存在，主要存放小程序的逻辑代码
  - **app.json：**必须存在，小程序配置文件，主要存放小程序的公共配置
  - **app.wxss:**  非必须存在，主要存放小程序公共样式表，类似于前端的CSS样式

![image-20251026174618379](https://raw.githubusercontent.com/calendar0917/images/master/image-20251026174618379.png)

- 每个小程序页面主要由四个文件组成：

  - **js文件：**必须存在，存放页面业务逻辑代码，编写的js代码。

    **wxml文件：**必须存在，存放页面结构，主要是做页面布局，页面效果展示的，类似于HTML页面。

    **json文件：**非必须，存放页面相关的配置。

    **wxss文件：**非必须，存放页面样式表，相当于CSS文件。

### 实现微信登录

- **步骤分析：**

  1. 小程序端，调用wx.login()获取code，就是授权码。

  2. 小程序端，调用wx.request()发送请求并携带code，请求开发者服务器(自己编写的后端服务)。

  3. 开发者服务端，通过HttpClient向微信接口服务发送请求，并携带appId+appsecret+code三个参数。

  4. 开发者服务端，接收微信接口服务返回的数据，session_key+opendId等。opendId是微信用户的唯一标识。

  5. 开发者服务端，自定义登录态，生成令牌(token)和openid等数据返回给小程序端，方便后绪请求身份校验。

  6. 小程序端，收到自定义登录态，存储storage。

  7. 小程序端，后绪通过wx.request()发起业务请求时，携带token。

  8. 开发者服务端，收到请求后，通过携带的token，解析当前登录用户的id。

  9. 开发者服务端，身份校验通过后，继续相关的业务逻辑处理，最终返回业务数据。

- 基于**文档**描述的所需参数，即可设计出登录接口

  - 请求参数：前端像后端传递 code

  - 返回数据：后端请求接口，再返回 id、openid、token

- 设计用户表

- Controller

```java
public class UserController {

    @Autowired
    private UserService userService;
    @Autowired
    private JwtProperties jwtProperties;
    
    /**
     * 微信登录
     * @param userLoginDTO
     * @return
     */
    @PostMapping("/login")
    @ApiOperation("微信登录")
    public Result<UserLoginVO> login(@RequestBody UserLoginDTO userLoginDTO){
        log.info("微信用户登录：{}",userLoginDTO.getCode());

        //微信登录
        User user = userService.wxLogin(userLoginDTO);

        //为微信用户生成jwt令牌
        Map<String, Object> claims = new HashMap<>();
        claims.put(JwtClaimsConstant.USER_ID,user.getId());
        String token = JwtUtil.createJWT(jwtProperties.getUserSecretKey(), jwtProperties.getUserTtl(), claims);

        UserLoginVO userLoginVO = UserLoginVO.builder()
                .id(user.getId())
                .openid(user.getOpenid())
                .token(token)
                .build();
        return Result.success(userLoginVO);
    }
}
```

- Service

```java
package com.sky.service.impl;
@Service
@Slf4j
public class UserServiceImpl implements UserService {

    //微信服务接口地址
    public static final String WX_LOGIN = "https://api.weixin.qq.com/sns/jscode2session";

    @Autowired
    private WeChatProperties weChatProperties;
    @Autowired
    private UserMapper userMapper;

    /**
     * 微信登录
     * @param userLoginDTO
     * @return
     */
    public User wxLogin(UserLoginDTO userLoginDTO) {
        String openid = getOpenid(userLoginDTO.getCode());

        //判断openid是否为空，如果为空表示登录失败，抛出业务异常
        if(openid == null){
            throw new LoginFailedException(MessageConstant.LOGIN_FAILED);
        }

        //判断当前用户是否为新用户
        User user = userMapper.getByOpenid(openid);

        //如果是新用户，自动完成注册
        if(user == null){
            user = User.builder()
                    .openid(openid)
                    .createTime(LocalDateTime.now())
                    .build();
            userMapper.insert(user);//后绪步骤实现
        }

        //返回这个用户对象
        return user;
    }

    /**
     * 调用微信接口服务，获取微信用户的openid
     * @param code
     * @return
     */
    private String getOpenid(String code){
        //调用微信接口服务，获得当前微信用户的openid
        Map<String, String> map = new HashMap<>();
        map.put("appid",weChatProperties.getAppid());
        map.put("secret",weChatProperties.getSecret());
        map.put("js_code",code);
        map.put("grant_type","authorization_code");
        String json = HttpClientUtil.doGet(WX_LOGIN, map);

        JSONObject jsonObject = JSON.parseObject(json);
        String openid = jsonObject.getString("openid");
        return openid;
    }
}
```

- 配置令牌校验
  - 学一下怎么校验
  - `Claims claims = JwtUtil.parseJWT(jwtProperties.getUserSecretKey(), token);`

```java
@Component
@Slf4j
public class JwtTokenUserInterceptor implements HandlerInterceptor {

    @Autowired
    private JwtProperties jwtProperties;

    /**
     * 校验jwt
     *
     * @param request
     * @param response
     * @param handler
     * @return
     * @throws Exception
     */
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        //判断当前拦截到的是Controller的方法还是其他资源
        if (!(handler instanceof HandlerMethod)) {
            //当前拦截到的不是动态方法，直接放行
            return true;
        }

        //1、从请求头中获取令牌
        String token = request.getHeader(jwtProperties.getUserTokenName());

        //2、校验令牌
        try {
            log.info("jwt校验:{}", token);
            Claims claims = JwtUtil.parseJWT(jwtProperties.getUserSecretKey(), token);
            Long userId = Long.valueOf(claims.get(JwtClaimsConstant.USER_ID).toString());
            log.info("当前用户的id：", userId);
            BaseContext.setCurrentId(userId);
            //3、通过，放行
            return true;
        } catch (Exception ex) {
            //4、不通过，响应401状态码
            response.setStatus(401);
            return false;
        }
    }
}
```

## 缓存

### 菜品 - 原始方法

> 用户端小程序展示的菜品数据都是通过查询数据库获得，如果用户端访问量比较大，数据库访问压力随之增大。
>
> - 用 Redis 缓存菜品数据，减少数据库操作

- **缓存逻辑分析：**
  - 每个分类下的菜品保存一份缓存数据
  - 数据库中菜品数据有变更时清理缓存数据

- Controller

```java
// 修改 DishController 调用 Mapper 的逻辑
public Result<List<DishVO>> list(Long categoryId) {

        //构造redis中的key，规则：dish_分类id
        String key = "dish_" + categoryId;

        //查询redis中是否存在菜品数据
        List<DishVO> list = (List<DishVO>) redisTemplate.opsForValue().get(key);
        if(list != null && list.size() > 0){
            //如果存在，直接返回，无须查询数据库
            return Result.success(list);
        }
		////////////////////////////////////////////////////////
        Dish dish = new Dish();
        dish.setCategoryId(categoryId);
        dish.setStatus(StatusConstant.ENABLE);//查询起售中的菜品

        //如果不存在，查询数据库，将查询到的数据放入redis中
        list = dishService.listWithFlavor(dish);
        ////////////////////////////////////////////////////////
        redisTemplate.opsForValue().set(key, list);

        return Result.success(list);
    }
```

> 为了保证**数据库**和**Redis**中的数据保持一致，修改**管理端接口 DishController** 的相关方法，加入清理缓存逻辑。

- Controller
  - 再新增、删除、更新等逻辑后面添加
  - 支持通配符删除

```java
// 管理端 DishController 中的清理缓存逻辑
    private void cleanCache(String pattern){
        Set keys = redisTemplate.keys(pattern);
        redisTemplate.delete(keys);
    }
```

### 套餐 - SpringCache

> Spring Cache 是一个框架，实现了基于注解的缓存功能，只需要简单地加一个注解，就能实现缓存功能。
>
> Spring Cache 提供了一层抽象，底层可以切换不同的缓存实现，例如：
>
> - EHCache
> - Caffeine
> - Redis(常用)
>
> ```xml
> <dependency>
> 	<groupId>org.springframework.boot</groupId>
> 	<artifactId>spring-boot-starter-cache</artifactId>  		            		       	 <version>2.7.3</version> 
> </dependency>
> ```

- 常用注解

| **注解**       | **说明**                                                     |
| -------------- | ------------------------------------------------------------ |
| @EnableCaching | 开启缓存注解功能，通常加在启动类上                           |
| @Cacheable     | 在方法执行前先查询缓存中是否有数据，如果有数据，则直接返回缓存数据；如果没有缓存数据，调用方法并将方法返回值放到缓存中 |
| @CachePut      | 将方法的返回值放到缓存中                                     |
| @CacheEvict    | 将一条或多条数据从缓存中删除                                 |

> @CachePut(value = "userCache", key = "#user.id")
>
> @Cacheable(cacheNames = "userCache",key="#id")
>
> @CacheEvict(cacheNames = "userCache",key = "#id")

- 启动类中添加 `@EnableCaching`
- 用户端 SetmealController 的 list 方法上加入@Cacheable注解

`@Cacheable(cacheNames = "setmealCache",key = "#categoryId")`

- 管理端接口SetmealController的 save、delete、update、startOrStop等方法上加入 CacheEvict 注解

`@CacheEvict(cacheNames = "setmealCache",allEntries = true)`

## 购物车

### 新增

- 关注点
  - 只能查询自己购物车的数据：`shoppingCart.setUserId(BaseContext.getCurrentId());`
  - 添加菜品 or 套餐的判断

- Controller

```java
package com.sky.service.impl;
@Service
public class ShoppingCartServiceImpl implements ShoppingCartService {

    @Autowired
    private ShoppingCartMapper shoppingCartMapper;
    @Autowired
    private DishMapper dishMapper;
    @Autowired
    private SetmealMapper setmealMapper;

    /**
     * 添加购物车
     *
     * @param shoppingCartDTO
     */
    public void addShoppingCart(ShoppingCartDTO shoppingCartDTO) {
        ShoppingCart shoppingCart = new ShoppingCart();
        BeanUtils.copyProperties(shoppingCartDTO, shoppingCart);
        //只能查询自己的购物车数据
        shoppingCart.setUserId(BaseContext.getCurrentId());

        //判断当前商品是否在购物车中
        List<ShoppingCart> shoppingCartList = shoppingCartMapper.list(shoppingCart);

        if (shoppingCartList != null && shoppingCartList.size() == 1) {
            //如果已经存在，就更新数量，数量加1
            shoppingCart = shoppingCartList.get(0);
            shoppingCart.setNumber(shoppingCart.getNumber() + 1);
            shoppingCartMapper.updateNumberById(shoppingCart);
        } else {
            //如果不存在，插入数据，数量就是1

            //判断当前添加到购物车的是菜品还是套餐
            Long dishId = shoppingCartDTO.getDishId();
            if (dishId != null) {
                //添加到购物车的是菜品
                Dish dish = dishMapper.getById(dishId);
                shoppingCart.setName(dish.getName());
                shoppingCart.setImage(dish.getImage());
                shoppingCart.setAmount(dish.getPrice());
            } else {
                //添加到购物车的是套餐
                Setmeal setmeal = setmealMapper.getById(shoppingCartDTO.getSetmealId());
                shoppingCart.setName(setmeal.getName());
                shoppingCart.setImage(setmeal.getImage());
                shoppingCart.setAmount(setmeal.getPrice());
            }
            shoppingCart.setNumber(1);
            shoppingCart.setCreateTime(LocalDateTime.now());
            shoppingCartMapper.insert(shoppingCart);
        }
    }
}
```

- Service

```java
@Service
public class ShoppingCartServiceImpl implements ShoppingCartService {

    @Autowired
    private ShoppingCartMapper shoppingCartMapper;
    @Autowired
    private DishMapper dishMapper;
    @Autowired
    private SetmealMapper setmealMapper;

    /**
     * 添加购物车
     *
     * @param shoppingCartDTO
     */
    public void addShoppingCart(ShoppingCartDTO shoppingCartDTO) {
        ShoppingCart shoppingCart = new ShoppingCart();
        BeanUtils.copyProperties(shoppingCartDTO, shoppingCart);
        //只能查询自己的购物车数据
        shoppingCart.setUserId(BaseContext.getCurrentId());

        //判断当前商品是否在购物车中
        List<ShoppingCart> shoppingCartList = shoppingCartMapper.list(shoppingCart);

        if (shoppingCartList != null && shoppingCartList.size() == 1) {
            //如果已经存在，就更新数量，数量加1
            shoppingCart = shoppingCartList.get(0);
            shoppingCart.setNumber(shoppingCart.getNumber() + 1);
            shoppingCartMapper.updateNumberById(shoppingCart);
        } else {
            //如果不存在，插入数据，数量就是1

            //判断当前添加到购物车的是菜品还是套餐
            Long dishId = shoppingCartDTO.getDishId();
            if (dishId != null) {
                //添加到购物车的是菜品
                Dish dish = dishMapper.getById(dishId);
                shoppingCart.setName(dish.getName());
                shoppingCart.setImage(dish.getImage());
                shoppingCart.setAmount(dish.getPrice());
            } else {
                //添加到购物车的是套餐
                Setmeal setmeal = setmealMapper.getById(shoppingCartDTO.getSetmealId());
                shoppingCart.setName(setmeal.getName());
                shoppingCart.setImage(setmeal.getImage());
                shoppingCart.setAmount(setmeal.getPrice());
            }
            shoppingCart.setNumber(1);
            shoppingCart.setCreateTime(LocalDateTime.now());
            shoppingCartMapper.insert(shoppingCart);
        }
    }
}
```

- Mapper

```java
package com.sky.mapper;
@Mapper
public interface ShoppingCartMapper {
    /**
     * 条件查询
     *
     * @param shoppingCart
     * @return
     */
    List<ShoppingCart> list(ShoppingCart shoppingCart);

    /**
     * 更新商品数量
     *
     * @param shoppingCart
     */
    @Update("update shopping_cart set number = #{number} where id = #{id}")
    void updateNumberById(ShoppingCart shoppingCart);

    /**
     * 插入购物车数据
     *
     * @param shoppingCart
     */
    @Insert("insert into shopping_cart (name, user_id, dish_id, setmeal_id, dish_flavor, number, amount, image, create_time) " +
            " values (#{name},#{userId},#{dishId},#{setmealId},#{dishFlavor},#{number},#{amount},#{image},#{createTime})")
    void insert(ShoppingCart shoppingCart);
}
```

> - 注意！User 需要用到 id，故需要设置回填！！
>
> `useGeneratedKeys`
>
> ```xml
> <insert id="insert" useGeneratedKeys="true" keyProperty="id">
>     insert into user (openid, name, phone, sex, id_number, avatar, create_time)
>     values (#{openid}, #{name}, #{phone}, #{sex}, #{idNumber}, #{avatar}, #{createTime})
> </insert>
> ```

> 注意！定义了 jwt 拦截器以后要在配置类（`WebMvcConfuguration`）里注册！！
>
> ```java
> protected void addInterceptors(InterceptorRegistry registry) {
>     log.info("开始注册自定义拦截器...");
>     registry.addInterceptor(jwtTokenAdminInterceptor)
>             .addPathPatterns("/admin/**")
>             .excludePathPatterns("/admin/employee/login");
>     registry.addInterceptor(jwtTokenUserInterceptor)
>             .addPathPatterns("/user/**")
>             .excludePathPatterns("/user/user/login");
> }
> ```

### 查询

类似，调用动态查询即可

### 删除

略

## 地址簿

> **接口设计：**
>
> - 新增地址
> - 查询登录用户所有地址
> - 查询默认地址
> - 根据id修改地址
> - 根据id删除地址
> - 根据id查询地址
> - 设置默认地址

类似的增删改查

## 用户下单

- 根据接口设计 DTO、VO
- Controller

```java
package com.sky.controller.user;

/**
 * 订单
 */
@RestController("userOrderController")
@RequestMapping("/user/order")
@Slf4j
@Api(tags = "C端-订单接口")
public class OrderController {

    @Autowired
    private OrderService orderService;
    @PostMapping("/submit")
    @ApiOperation("用户下单")
    public Result<OrderSubmitVO> submit(@RequestBody OrdersSubmitDTO ordersSubmitDTO) {
        log.info("用户下单：{}", ordersSubmitDTO);
        OrderSubmitVO orderSubmitVO = orderService.submitOrder(ordersSubmitDTO);
        return Result.success(orderSubmitVO);
    } 

}
```

- Service
  - 异常处理
  - 查询购物车数据
  - 构造、添加订单表
  - 构造、加入订单明细表
  - 清空购物车，封装返回结果

```java
package com.sky.service.impl;

/**
 * 订单
 */
@Service
@Slf4j
public class OrderServiceImpl implements OrderService {
    @Autowired
    private OrderMapper orderMapper;
    @Autowired
    private OrderDetailMapper orderDetailMapper;
    @Autowired
    private ShoppingCartMapper shoppingCartMapper;
    @Autowired
    private AddressBookMapper addressBookMapper;


    /**
     * 用户下单
     *
     * @param ordersSubmitDTO
     * @return
     */
    @Transactional
    public OrderSubmitVO submitOrder(OrdersSubmitDTO ordersSubmitDTO) {
        //异常情况的处理（收货地址为空、超出配送范围、购物车为空）
        AddressBook addressBook = addressBookMapper.getById(ordersSubmitDTO.getAddressBookId());
        if (addressBook == null) {
            throw new AddressBookBusinessException(MessageConstant.ADDRESS_BOOK_IS_NULL);
        }

        Long userId = BaseContext.getCurrentId();
        ShoppingCart shoppingCart = new ShoppingCart();
        shoppingCart.setUserId(userId);

        //查询当前用户的购物车数据
        List<ShoppingCart> shoppingCartList = shoppingCartMapper.list(shoppingCart);
        if (shoppingCartList == null || shoppingCartList.size() == 0) {
            throw new ShoppingCartBusinessException(MessageConstant.SHOPPING_CART_IS_NULL);
        }

        //构造订单数据
        Orders order = new Orders();
        BeanUtils.copyProperties(ordersSubmitDTO,order);
        order.setPhone(addressBook.getPhone());
        order.setAddress(addressBook.getDetail());
        order.setConsignee(addressBook.getConsignee());
        order.setNumber(String.valueOf(System.currentTimeMillis()));
        order.setUserId(userId);
        order.setStatus(Orders.PENDING_PAYMENT);
        order.setPayStatus(Orders.UN_PAID);
        order.setOrderTime(LocalDateTime.now());

        //向订单表插入1条数据
        orderMapper.insert(order);

        //订单明细数据
        List<OrderDetail> orderDetailList = new ArrayList<>();
        for (ShoppingCart cart : shoppingCartList) {
            OrderDetail orderDetail = new OrderDetail();
            BeanUtils.copyProperties(cart, orderDetail);
            orderDetail.setOrderId(order.getId());
            orderDetailList.add(orderDetail);
        }

        //向明细表插入n条数据
        orderDetailMapper.insertBatch(orderDetailList);

        //清理购物车中的数据
        shoppingCartMapper.deleteByUserId(userId);

        //封装返回结果
        OrderSubmitVO orderSubmitVO = OrderSubmitVO.builder()
                .id(order.getId())
                .orderNumber(order.getNumber())
                .orderAmount(order.getAmount())
                .orderTime(order.getOrderTime())
                .build();

        return orderSubmitVO;
    }
    
}
```

## 订单支付

> 参考支付文档：https://pay.weixin.qq.com/static/product/product_index.shtml

- 流程：
  - 关键是新增了微信后台的接口

![image-20251027082949327](https://raw.githubusercontent.com/calendar0917/images/master/image-20251027082949327.png)

## 订单管理

### 查询历史订单

- 写代码之前先想清楚用什么实体类、查什么表什么字段、返回什么！
  - 订单表、订单明细表不同 --> 需要先查出订单 id，在去订单明细表中查明细

- 各个功能的接口要看清楚

- 接收参数不一定要用实体类接……
  - 还要注意实体类之间的继承关系

### 删除订单

- 业务逻辑、场景要考虑全面

> - 待支付和待接单状态下，用户可直接取消订单
> - 商家已接单状态下，用户取消订单需电话沟通商家
> - 派送中状态下，用户取消订单需电话沟通商家
> - 如果在待接单状态下取消订单，需要给用户退款
> - 取消订单后需要将订单状态修改为“已取消”

- 参考 Service

```java
	/**
     * 用户取消订单
     *
     * @param id
     */
    public void userCancelById(Long id) throws Exception {
        // 根据id查询订单
        Orders ordersDB = orderMapper.getById(id);

        // 校验订单是否存在
        if (ordersDB == null) {
            throw new OrderBusinessException(MessageConstant.ORDER_NOT_FOUND);
        }

        //订单状态 1待付款 2待接单 3已接单 4派送中 5已完成 6已取消
        if (ordersDB.getStatus() > 2) {
            throw new OrderBusinessException(MessageConstant.ORDER_STATUS_ERROR);
        }

        Orders orders = new Orders();
        orders.setId(ordersDB.getId());

        // 订单处于待接单状态下取消，需要进行退款
        if (ordersDB.getStatus().equals(Orders.TO_BE_CONFIRMED)) {
            //调用微信支付退款接口
            weChatPayUtil.refund(
                    ordersDB.getNumber(), //商户订单号
                    ordersDB.getNumber(), //商户退款单号
                    new BigDecimal(0.01),//退款金额，单位 元
                    new BigDecimal(0.01));//原订单金额

            //支付状态修改为 退款
            orders.setPayStatus(Orders.REFUND);
        }

        // 更新订单状态、取消原因、取消时间
        orders.setStatus(Orders.CANCELLED);
        orders.setCancelReason("用户取消");
        orders.setCancelTime(LocalDateTime.now());
        orderMapper.update(orders);
    }
```

### 再来一单

> 再来一单就是将原订单中的商品重新加入到购物车中

- Service
  - 注意看对象转换的 stream 流

```java
	/**
     * 再来一单
     *
     * @param id
     */
    public void repetition(Long id) {
        // 查询当前用户id
        Long userId = BaseContext.getCurrentId();

        // 根据订单id查询当前订单详情
        List<OrderDetail> orderDetailList = orderDetailMapper.getByOrderId(id);

        // 将订单详情对象转换为购物车对象
        List<ShoppingCart> shoppingCartList = orderDetailList.stream().map(x -> {
            ShoppingCart shoppingCart = new ShoppingCart();

            // 将原订单详情里面的菜品信息重新复制到购物车对象中
            BeanUtils.copyProperties(x, shoppingCart, "id");
            shoppingCart.setUserId(userId);
            shoppingCart.setCreateTime(LocalDateTime.now());

            return shoppingCart;
        }).collect(Collectors.toList());

        // 将购物车对象批量添加到数据库
        shoppingCartMapper.insertBatch(shoppingCartList);
    }
```

### 商家搜索

- SQL 中，`>=` 会和标签混淆，故要用 `&gt;` 代替

### 接单、拒单、派送

- 简化为修改 status 字段，并且要处理退款逻辑

- Service

```java
	/**
     * 拒单
     *
     * @param ordersRejectionDTO
     */
    public void rejection(OrdersRejectionDTO ordersRejectionDTO) throws Exception {
        // 根据id查询订单
        Orders ordersDB = orderMapper.getById(ordersRejectionDTO.getId());

        // 订单只有存在且状态为2（待接单）才可以拒单
        if (ordersDB == null || !ordersDB.getStatus().equals(Orders.TO_BE_CONFIRMED)) {
            throw new OrderBusinessException(MessageConstant.ORDER_STATUS_ERROR);
        }

        //支付状态
        Integer payStatus = ordersDB.getPayStatus();
        if (payStatus == Orders.PAID) {
            //用户已支付，需要退款
            String refund = weChatPayUtil.refund(
                    ordersDB.getNumber(),
                    ordersDB.getNumber(),
                    new BigDecimal(0.01),
                    new BigDecimal(0.01));
            log.info("申请退款：{}", refund);
        }

        // 拒单需要退款，根据订单id更新订单状态、拒单原因、取消时间
        Orders orders = new Orders();
        orders.setId(ordersDB.getId());
        orders.setStatus(Orders.CANCELLED);
        orders.setRejectionReason(ordersRejectionDTO.getRejectionReason());
        orders.setCancelTime(LocalDateTime.now());

        orderMapper.update(orders);
    }
```

## 距离检测

调用百度地图的接口，读文档即可

设置地址、规划路径、判断

- 添加到 ServiceImpl

```java
    @Value("${sky.shop.address}")
    private String shopAddress;

    @Value("${sky.baidu.ak}")
    private String ak;
```

```java
/**
     * 检查客户的收货地址是否超出配送范围
     * @param address
     */
    private void checkOutOfRange(String address) {
        Map map = new HashMap();
        map.put("address",shopAddress);
        map.put("output","json");
        map.put("ak",ak);

        //获取店铺的经纬度坐标
        String shopCoordinate = HttpClientUtil.doGet("https://api.map.baidu.com/geocoding/v3", map);

        JSONObject jsonObject = JSON.parseObject(shopCoordinate);
        if(!jsonObject.getString("status").equals("0")){
            throw new OrderBusinessException("店铺地址解析失败");
        }

        //数据解析
        JSONObject location = jsonObject.getJSONObject("result").getJSONObject("location");
        String lat = location.getString("lat");
        String lng = location.getString("lng");
        //店铺经纬度坐标
        String shopLngLat = lat + "," + lng;

        map.put("address",address);
        //获取用户收货地址的经纬度坐标
        String userCoordinate = HttpClientUtil.doGet("https://api.map.baidu.com/geocoding/v3", map);

        jsonObject = JSON.parseObject(userCoordinate);
        if(!jsonObject.getString("status").equals("0")){
            throw new OrderBusinessException("收货地址解析失败");
        }

        //数据解析
        location = jsonObject.getJSONObject("result").getJSONObject("location");
        lat = location.getString("lat");
        lng = location.getString("lng");
        //用户收货地址经纬度坐标
        String userLngLat = lat + "," + lng;

        map.put("origin",shopLngLat);
        map.put("destination",userLngLat);
        map.put("steps_info","0");

        //路线规划
        String json = HttpClientUtil.doGet("https://api.map.baidu.com/directionlite/v1/driving", map);

        jsonObject = JSON.parseObject(json);
        if(!jsonObject.getString("status").equals("0")){
            throw new OrderBusinessException("配送路线规划失败");
        }

        //数据解析
        JSONObject result = jsonObject.getJSONObject("result");
        JSONArray jsonArray = (JSONArray) result.get("routes");
        Integer distance = (Integer) ((JSONObject) jsonArray.get(0)).get("distance");

        if(distance > 5000){
            //配送距离超过5000米
            throw new OrderBusinessException("超出配送范围");
        }
    }
```

- 然后在 OrderServiceImpl 的 submitOrder 方法中调用上面的校验方法即可

## SpringTask - 订单计时

> Spring框架提供的任务调度工具，可以按照约定的时间自动执行某个代码逻辑。

### Cron 表达式

**cron表达式**其实就是一个字符串，通过cron表达式可以**定义任务触发的时间**

**构成规则：**分为6或7个域，由空格分隔开，每个域代表一个含义

每个域的含义分别为：秒、分钟、小时、日、月、周、年(可选)

**举例：**2022年10月12日上午9点整 对应的cron表达式为：**0 0 9 12 10 ? 2022**

**说明：**一般**日**和**周**的值不同时设置，其中一个设置，另一个用？表示。

> **通配符：**
>
> \* 表示所有值； 
>
> ? 表示未说明的值，即不关心它为何值； 
>
> \- 表示一个指定的范围； 
>
> , 表示附加一个可能值； 
>
> / 符号前表示开始时间，符号后表示每次递增的值；

- 使用步骤
  1. 导入 maven 坐标，spring-context
  2. 启动类添加注解 @EnableScheduling 开启任务调度
  3. 自定义定时任务类
- 定时任务类

```java
package com.sky.task;
@Component
@Slf4j
public class OrderTask {

    @Autowired
    private OrderMapper orderMapper;

    /**
     * 处理支付超时订单
     */
    @Scheduled(cron = "0 * * * * ?")
    public void processTimeoutOrder(){
        log.info("处理支付超时订单：{}", new Date());

        LocalDateTime time = LocalDateTime.now().plusMinutes(-15);

        // select * from orders where status = 1 and order_time < 当前时间-15分钟
        List<Orders> ordersList = orderMapper.getByStatusAndOrdertimeLT(Orders.PENDING_PAYMENT, time);
        if(ordersList != null && ordersList.size() > 0){
            ordersList.forEach(order -> {
                order.setStatus(Orders.CANCELLED);
                order.setCancelReason("支付超时，自动取消");
                order.setCancelTime(LocalDateTime.now());
                orderMapper.update(order);
            });
        }
    }
    /**
     * 处理“派送中”状态的订单
     */
    @Scheduled(cron = "0 0 1 * * ?")
    public void processDeliveryOrder(){
        log.info("处理派送中订单：{}", new Date());
        // select * from orders where status = 4 and order_time < 当前时间-1小时
        LocalDateTime time = LocalDateTime.now().plusMinutes(-60);
        List<Orders> ordersList = orderMapper.getByStatusAndOrdertimeLT(Orders.DELIVERY_IN_PROGRESS, time);

        if(ordersList != null && ordersList.size() > 0){
            ordersList.forEach(order -> {
                order.setStatus(Orders.COMPLETED);
                orderMapper.update(order);
            });
        }
    }
}
```

- Mapper

```java
	//根据状态和下单时间查询订单
    @Select("select * from orders where status = #{status} and order_time < #{orderTime}")
    List<Orders> getByStatusAndOrdertimeLT(Integer status, LocalDateTime orderTime);
```

## Websocket

> WebSocket 是**基于 TCP** 的一种新的**网络协议**。它实现了浏览器与服务器全双工通信——浏览器和服务器只需要完成一次握手，两者之间就可以创建**持久性**的连接， 并进行**双向**数据传输。
>
> **WebSocket缺点：**
>
> 1. 服务器长期维护长连接需要一定的成本
> 2. 各个浏览器支持程度不一
> 3. WebSocket 是长连接，受网络限制比较大，需要处理好重连
>
> **结论：**WebSocket并不能完全取代HTTP，它只适合在特定的场景下使用

- maven 坐标

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

- 服务组件定义

```java
package com.sky.websocket;

/**
 * WebSocket服务
 */
@Component
@ServerEndpoint("/ws/{sid}")
public class WebSocketServer {

    //存放会话对象
    private static Map<String, Session> sessionMap = new HashMap();

    /**
     * 连接建立成功调用的方法
     */
    @OnOpen
    public void onOpen(Session session, @PathParam("sid") String sid) {
        System.out.println("客户端：" + sid + "建立连接");
        sessionMap.put(sid, session);
    }

    /**
     * 收到客户端消息后调用的方法
     *
     * @param message 客户端发送过来的消息
     */
    @OnMessage
    public void onMessage(String message, @PathParam("sid") String sid) {
        System.out.println("收到来自客户端：" + sid + "的信息:" + message);
    }

    /**
     * 连接关闭调用的方法
     *
     * @param sid
     */
    @OnClose
    public void onClose(@PathParam("sid") String sid) {
        System.out.println("连接断开:" + sid);
        sessionMap.remove(sid);
    }

    /**
     * 群发
     *
     * @param message
     */
    public void sendToAllClient(String message) {
        Collection<Session> sessions = sessionMap.values();
        for (Session session : sessions) {
            try {
                //服务器向客户端发送消息
                session.getBasicRemote().sendText(message);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

}
```

- 配置类
  - 用于注册 WebSocket 的服务端组件

```java
package com.sky.config;

/**
 * WebSocket配置类，用于注册WebSocket的Bean
 */
@Configuration
@EnableWebsocket
public class WebSocketConfiguration {

    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }

}
```

- Impl 里引用

```java
        orderMapper.update(orders);
        Map map = new HashMap();
        map.put("type", 1);//消息类型，1表示来单提醒
        map.put("orderId", orders.getId());
        map.put("content", "订单号：" + outTradeNo);

        //通过WebSocket实现来单提醒，向客户端浏览器推送消息
        webSocketServer.sendToAllClient(JSON.toJSONString(map));
```

### 催单

类似

WebSocket 不知道为什么连不上……

## Apache Echarts、数据统计

> Apache ECharts 是一款基于 Javascript 的数据可视化图表库，提供直观，生动，可交互，可个性化定制的数据可视化图表。

### 营业额统计

> **注意：**具体返回数据一般由前端来决定，前端展示图表，具体折现图对应数据是什么格式，是有固定的要求的。

- Controller
  - 注意：`@DateTimeFormat(pattern = "yyyy-MM-dd")` 指定格式

```java
package com.sky.controller.admin;

/**
 * 报表
 */
@RestController
@RequestMapping("/admin/report")
@Slf4j
@Api(tags = "统计报表相关接口")
public class ReportController {

    @Autowired
    private ReportService reportService;

    /**
     * 营业额数据统计
     */
    @GetMapping("/turnoverStatistics")
    @ApiOperation("营业额数据统计")
    public Result<TurnoverReportVO> turnoverStatistics(
            @DateTimeFormat(pattern = "yyyy-MM-dd")
                    LocalDate begin,
            @DateTimeFormat(pattern = "yyyy-MM-dd")
                    LocalDate end) {
        return Result.success(reportService.getTurnover(begin, end));
    }

}
```

- Service
  - 用 map 来传递参数

```java
package com.sky.service.impl;

@Service
@Slf4j
public class ReportServiceImpl implements ReportService {

    @Autowired
    private OrderMapper orderMapper;

    /**
     * 根据时间区间统计营业额
     * @param begin
     * @param end
     * @return
     */
    public TurnoverReportVO getTurnover(LocalDate begin, LocalDate end) {
        List<LocalDate> dateList = new ArrayList<>();
        dateList.add(begin);

        while (!begin.equals(end)){
            begin = begin.plusDays(1);//日期计算，获得指定日期后1天的日期
            dateList.add(begin);
        }

        List<Double> turnoverList = new ArrayList<>();
        for (LocalDate date : dateList) {
            LocalDateTime beginTime = LocalDateTime.of(date, LocalTime.MIN);
            LocalDateTime endTime = LocalDateTime.of(date, LocalTime.MAX);
            Map map = new HashMap();
            map.put("status", Orders.COMPLETED);
            map.put("begin",beginTime);
            map.put("end", endTime);
            Double turnover = orderMapper.sumByMap(map);
            turnover = turnover == null ? 0.0 : turnover;
            turnoverList.add(turnover);
        }

        //数据封装
        return TurnoverReportVO.builder()
                .dateList(StringUtils.join(dateList,","))
                .turnoverList(StringUtils.join(turnoverList,","))
                .build();
    }
}
```

- Mapper
  - 接收 map 直接取就可以

```xml
    <select id="sumByMap" resultType="java.lang.Double">
        select sum(amount) from orders
        <where>
            <if test="status != null">
                and status = #{status}
            </if>
            <if test="begin != null">
                and order_time &gt;= #{begin}
            </if>
            <if test="end != null">
                and order_time &lt;= #{end}
            </if>
        </where>
    </select>
```

### 用户统计

- Service
  - 自定义方法来计算总数
  - 返回的是字符串，还要进行处理

```java
@Override
public UserReportVO getUserStatistics(LocalDate begin, LocalDate end) {
    List<LocalDate> dateList = new ArrayList<>();
    dateList.add(begin);

    while (!begin.equals(end)){
        begin = begin.plusDays(1);
        dateList.add(begin);
    }
    List<Integer> newUserList = new ArrayList<>(); //新增用户数
    List<Integer> totalUserList = new ArrayList<>(); //总用户数

    for (LocalDate date : dateList) {
        LocalDateTime beginTime = LocalDateTime.of(date, LocalTime.MIN);
        LocalDateTime endTime = LocalDateTime.of(date, LocalTime.MAX);
        //新增用户数量 select count(id) from user where create_time > ? and create_time < ?
        Integer newUser = getUserCount(beginTime, endTime);
        //总用户数量 select count(id) from user where  create_time < ?
        Integer totalUser = getUserCount(null, endTime);

        newUserList.add(newUser);
        totalUserList.add(totalUser);
    }

    return UserReportVO.builder()
            .dateList(StringUtils.join(dateList,","))
            .newUserList(StringUtils.join(newUserList,","))
            .totalUserList(StringUtils.join(totalUserList,","))
            .build();
}

/**
 * 根据时间区间统计用户数量
 * @param beginTime
 * @param endTime
 * @return
 */
private Integer getUserCount(LocalDateTime beginTime, LocalDateTime endTime) {
    Map map = new HashMap();
    map.put("begin",beginTime);
    map.put("end", endTime);
    return userMapper.countByMap(map);
}
```

- Mapper

```xml
<select id="countByMap" resultType="java.lang.Integer">
        select count(id) from user
        <where>
            <if test="begin != null">
                and create_time &gt;= #{begin}
            </if>
            <if test="end != null">
                and create_time &lt;= #{end}
            </if>
        </where>
</select>
```

### 订单统计

类似

### 销量排行榜

- Mapper
  - 只返回前十条数据即可，用 limit 限制

```xml
<select id="getSalesTop10" resultType="com.sky.dto.GoodsSalesDTO">
    select od.name name,sum(od.number) number from order_detail od ,orders o
    where od.order_id = o.id
    and o.status = 5
    <if test="begin != null">
        and order_time &gt;= #{begin}
    </if>
    <if test="end != null">
        and order_time &lt;= #{end}
    </if>
    group by name
    order by number desc
    limit 0, 10
</select>
```

## Apache POI

> Apache POI 是一个处理Miscrosoft Office各种文件格式的开源项目。简单来说就是，我们可以使用 POI 在 Java 程序中对Miscrosoft Office各种文件进行读写操作。
> 一般情况下，POI 都是用于操作 Excel 文件。

- 坐标

```xml
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi</artifactId>
    <version>3.16</version>
</dependency>
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi-ooxml</artifactId>
    <version>3.16</version>
</dependency>
```

- 数据处理演示

```java
package com.sky.test;

import org.apache.poi.xssf.usermodel.XSSFCell;
import org.apache.poi.xssf.usermodel.XSSFRow;
import org.apache.poi.xssf.usermodel.XSSFSheet;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;

public class POITest {

    /**
     * 基于POI向Excel文件写入数据
     * @throws Exception
     */
    public static void write() throws Exception{
        //在内存中创建一个Excel文件对象
        XSSFWorkbook excel = new XSSFWorkbook();
        //创建Sheet页
        XSSFSheet sheet = excel.createSheet("itcast");

        //在Sheet页中创建行，0表示第1行
        XSSFRow row1 = sheet.createRow(0);
        //创建单元格并在单元格中设置值，单元格编号也是从0开始，1表示第2个单元格
        row1.createCell(1).setCellValue("姓名");
        row1.createCell(2).setCellValue("城市");

        XSSFRow row2 = sheet.createRow(1);
        row2.createCell(1).setCellValue("张三");
        row2.createCell(2).setCellValue("北京");

        XSSFRow row3 = sheet.createRow(2);
        row3.createCell(1).setCellValue("李四");
        row3.createCell(2).setCellValue("上海");

        FileOutputStream out = new FileOutputStream(new File("D:\\itcast.xlsx"));
        //通过输出流将内存中的Excel文件写入到磁盘上
        excel.write(out);

        //关闭资源
        out.flush();
        out.close();
        excel.close();
    }
    public static void main(String[] args) throws Exception {
        write();
    }
}
```

- 数据读取演示

```java
package com.sky.test;

import org.apache.poi.xssf.usermodel.XSSFCell;
import org.apache.poi.xssf.usermodel.XSSFRow;
import org.apache.poi.xssf.usermodel.XSSFSheet;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;

public class POITest {
    /**
     * 基于POI读取Excel文件
     * @throws Exception
     */
    public static void read() throws Exception{
        FileInputStream in = new FileInputStream(new File("D:\\itcast.xlsx"));
        //通过输入流读取指定的Excel文件
        XSSFWorkbook excel = new XSSFWorkbook(in);
        //获取Excel文件的第1个Sheet页
        XSSFSheet sheet = excel.getSheetAt(0);

        //获取Sheet页中的最后一行的行号
        int lastRowNum = sheet.getLastRowNum();

        for (int i = 0; i <= lastRowNum; i++) {
            //获取Sheet页中的行
            XSSFRow titleRow = sheet.getRow(i);
            //获取行的第2个单元格
            XSSFCell cell1 = titleRow.getCell(1);
            //获取单元格中的文本内容
            String cellValue1 = cell1.getStringCellValue();
            //获取行的第3个单元格
            XSSFCell cell2 = titleRow.getCell(2);
            //获取单元格中的文本内容
            String cellValue2 = cell2.getStringCellValue();

            System.out.println(cellValue1 + " " +cellValue2);
        }

        //关闭资源
        in.close();
        excel.close();
    }

    public static void main(String[] args) throws Exception {
        read();
    }
}
```

后续代码类似，要用到的时候再学吧，完结。