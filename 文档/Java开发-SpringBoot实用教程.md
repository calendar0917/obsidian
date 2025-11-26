---
title: "SpringBoot"
subtitle: "介绍SpringBoot使用的文章"
summary: "简要介绍SpringBoot使用"
description: "简要介绍SpringBoot使用"
image: "https://raw.githubusercontent.com/calendar0917/images/master/20251012171949620.png"
date: 2025-10-12
lastmod: 2025-10-12
draft: false
toc:
 enable: true
weight: false
categories: ["开发"]
tags: ["技术"]
---

## SpringBoot

### 简介

设计目的：**简化Spring应用的初始搭建以及开发过程**

Spring 程序缺点

- 依赖设置繁琐
  - 去除 spring-web 和 spring-webmvc 坐标
- 配置繁琐

SpringBoot 核心功能及优点：

- 起步依赖（简化依赖配置）

- 自动配置
- 辅助功能（内置服务器）

#### parent 管理版本

由 **parent** 帮助开发者统一的进行各种技术的版本管理

- 只控制版本，不负责导入坐标

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.5.4</version>
</parent>
```

#### starter 依赖组合

 starter定义了使用某种技术时对于依赖的**固定搭配格式**，使用starter可以帮助开发者减少依赖配置。

#### 引导类

这个类在SpringBoot程序中是所有功能的入口，称为**引导类**，最典型的特征就是当前类上方声明了一个注解 `@SpringBootApplication。`

- 用于启动程序
- 创建并初始化 Spring 容器

#### 内嵌 Tomcat

整合到了：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <exclusions>
            <exclusion>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-tomcat</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
</dependencies>

```

### 基础配置

默认配置文件：`application.properties`，配置指定属性即可

1.  配置文件间的加载优先级 properties（最高）> yml > yaml（最低）
2. 不同配置文件中相同配置按照加载优先级相互覆盖，不同配置文件中不同配置全部保留

指定SpringBoot配置文件

- Setting → Project Structure → Facets
- 选中对应项目/工程
- Customize Spring Boot
- 选择配置文件

#### YML 数据读取

##### 使用Spring中的注解

@Value读取单个数据

```java
@Value("${server.port}")
private int port;
```

##### 使用默认配置类

SpringBoot 提供了一个对象，能够把所有的数据都封装到这一个对象中，这个对象叫做 Environment，使用自动装配注解可以将所有的yaml数据封装到这个对象中

```java
@Autowired
private Environment env;
...
env.getProperty("...");
```

##### 使用自定义配置类

SpringBoot 也提供了可以将一组 yaml 对象数据封装一个 Java 对象的操作

enterprise 指定加载某一组 yaml 配置

```java
@Component
@ConfigurationProperties(prefix = "enterprise")
public class Enterprise {
private String name;
private Integer age;
private String[] subject;
}
```

##### 数据引用

```yml
baseDir: /usr/local/fire
center:
    dataDir: ${baseDir}/data
    tmpDir: ${baseDir}/tmp
    logDir: ${baseDir}/log
    msgDir: ${baseDir}/msgDir
```

### SSMP 整合

#### JUnit

```java
@SpringBootTest(classes = Springboot04JunitApplication.class)
// @ContextConfiguration(classes = Springboot04JunitApplication.class)
class Springboot04JunitApplicationTests {
    //注入你要测试的对象
    @Autowired
    private BookDao bookDao;
    @Test
    void contextLoads() {
        //执行要测试的对象对应的方法
        bookDao.save();
        System.out.println("two...");
    }
}
```

#### Mybatis

在配置、引入依赖时已经整合

```java
@Mapper
public interface BookDao {
    @Select("select * from tbl_book where id = #{id}")
    public Book getById(Integer id);
}

```

```yml
#2.配置相关信息
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/ssm_db?serverTimezone=Asia/Shanghai
    username: root
    password: root
```

#### Mybatis-plus

需要用阿里云的 url 导入

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.4.3</version>
</dependency>
```

配置所有数据库表名的前缀名：

```yml
mybatis-plus:
  global-config:
    db-config:
      table-prefix: tbl_		#设置所有表的通用前缀名称为tbl_
```

### 其他

#### Lombok

简化POJO实体类开发，SpringBoot 目前默认集成了 lombok 技术

- 可以通过一个注解@Data完成一个实体类对应的getter，setter，toString，equals，hashCode等操作的快速添加

```xml
<dependencies>
    <!--lombok-->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
</dependencies>
```

