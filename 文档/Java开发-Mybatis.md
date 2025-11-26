---
title: "Mybatis"
subtitle: "记录 Mybatis 的学习"
summary: "记录 Mybatis 的学习"
description: "记录 Mybatis 的学习"
image: "https://raw.githubusercontent.com/calendar0917/images/master/20251012201626030.png"
date: 2025-10-12
lastmod: 2025-10-12
draft: false
toc:
 enable: true
weight: false
categories: ["开发"]
tags: ["技术"]
---

## Mybatis

### 介绍

- 轻量级，性能出色
- 封装 JDBC
- SQL 和 Java 编码分开，功能边界清晰。Java 代码专注业务、SQL 语句专注数据
- 官网：https://mybatis.org/mybatis-3/zh/#

### 使用

- 创建SpringBoot工程、引入Mybatis相关依赖
- 准备数据库表即对应实体类

- 配置 Mybatis（在`application.properties`中数据库连接信息）

编写 Mybatis 程序：编写 Mybatis 的持久层接口，定义 SQL（注解/XML）

- `@Mapper`：应用程序在运行时，会自动的为该接口创建一个实现类对象（代理对象），并且会自动将该实现类对象存入I0C容器

#### 辅助配置

- 定义注解的语句为 MySQL 语法

- IDEA 连接数据库

- 日志

```xml
#配置mybatis的日志输出
mybatis.configuration.logimpl=org.apache.ibatis.logging.stdout.StdoutImpl
```

#### 数据库连接池

- 数据库连接池是个容器，负责分配、管理数据库连接（Connection）
  - 资源重用
- 允许应用程序重复使用一个现有的数据库连接，而不是再重新建立一个
  - 提升系统响应速度
- 释放空闲时间超过最大空闲时间的连接，来避免因为没有释放连接而引起的数据库连接遗漏
  - 避免数据库连接遗漏

```xml
// 引入德鲁伊连接池
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.2.19</version>
</dependency>
```

`spring.datasource.type=com.alibaba.druid.pool.DruidDataSource`

底层实现 DataSource 接口

#### 操作

##### 新增

```java
@Insert("insert insert(username,password,name,age) values (#{username},#{password},#{name},#{age})") // 写的是对象属性名
public void insert(User user2);
```

##### 删除

```java
@Delete("delete from user where id = #{id}")
public Integer deleteById(Integer id);
// 可以返回影响的行数
```

> ![image-20251012210220307](https://raw.githubusercontent.com/calendar0917/images/master/image-20251012210220307.png)

##### 修改

```java
@Update("update user set username=#{username}, password=#{password}, name=#{name}, age=#{age} where id=#{id}")
public void update(User user);
```

##### 查询

```java
@Select("select * from user where username=#{username} and password=#{password}")
public User findByUsernameAndPassword(@Param("username") String username, @Param("password") String password);
```

- 有多个参数时，需要 `@param` 注解

##### XML 映射配置

默认规则：

1. XML 映射文件的名称与 Mapper 接口名称一致，并且将 XML 映射文件和 Mapper 接口放置在相同包下（ resource 中同包同名）。

2. XML 映射文件的 namespace 属性为 Mapper 接口全限定名一致。

```xml
<mapper namespace="com.itheima.mapper.UserMapper">
    <select id="findAll" resultType="com.itheima.pojo.User">
    	select id，username，password，name，age from user
    </select>
</mapper>
```



2. XML 映射文件中 sql 语句的 id 与 Mapper 接口中的方法名一致，并保持返回类型一致。

#### 动态 SQL

##### IF

```xml
<select id="selectAllWebsite" resultMap="myResult">
    select id,name,url from website where 1=1
    <if test="name != null">
        AND name like #{name}
    </if>
    <if test="url!= null">
        AND url like #{url}
    </if>
</select>
```

##### choose-when-otherwise

```xml
<mapper namespace="net.biancheng.mapper.WebsiteMapper">
    <select id="selectWebsite"
            parameterType="net.biancheng.po.Website"
            resultType="net.biancheng.po.Website">
        SELECT id,name,url,age,country
        FROM website WHERE 1=1
        <choose>
            <when test="name != null and name !=''">
                AND name LIKE CONCAT('%',#{name},'%')
            </when>
            <when test="url != null and url !=''">
                AND url LIKE CONCAT('%',#{url},'%')
            </when>
            <otherwise>
                AND age is not null
            </otherwise>
        </choose>
    </select>
</mapper>
```

> 注意：AND 不能省！

##### WHERE

where 会检索语句，它会将 where 后的第一个 SQL 条件语句的 AND 或者 OR 关键词去掉。

```xml
<select id="selectWebsite" resultType="net.biancheng.po.Website">
    select id,name,url from website
    <where>
        <if test="name != null">
            AND name like #{name}
        </if>
        <if test="url!= null">
            AND url like #{url}
        </if>
    </where>
</select>
```

##### SET

set 标签可以为 SQL 语句动态的添加 set 关键字，剔除追加到条件末尾多余的逗号

```xml
<update id="updateWebsite"
            parameterType="net.biancheng.po.Website">
        UPDATE website
        <set>
            <if test="name!=null">name=#{name}</if>
            <if test="url!=null">url=#{url}</if>
        </set>
        WHERE id=#{id}
   </update>
```

##### foreach

foreach 标签用于循环语句，它很好的支持了数据和 List、set 接口的集合，并对此提供遍历的功能

```xml
<foreach item="item" index="index" collection="list|array|map key" open="(" separator="," close=")">
    参数值
</foreach>
```

foreach 标签主要有以下属性，说明如下。

- item：表示集合中每一个元素进行迭代时的别名。
- index：指定一个名字，表示在迭代过程中每次迭代到的位置。
- open：表示该语句以什么开始（既然是 in 条件语句，所以必然以(开始）。
- separator：表示在每次进行迭代之间以什么符号作为分隔符（既然是 in 条件语句，所以必然以,作为分隔符）。
- close：表示该语句以什么结束（既然是 in 条件语句，所以必然以)开始）。

使用 foreach 标签时，最关键、最容易出错的是 collection 属性，该属性是必选的，但在不同情况下该属性的值是不一样的，主要有以下 3 种情况：

- 如果传入的是单参数且参数类型是一个 List，collection 属性值为 list。
- 如果传入的是单参数且参数类型是一个 array 数组，collection 的属性值为 array。
- 如果传入的参数是多个，需要把它们封装成一个 Map，当然单参数也可以封装成 Map。Map 的 key 是参数名，collection 属性值是传入的 List 或 array 对象在自己封装的 Map 中的 key。

##### trim

trim 一般用于去除 SQL 语句中多余的 AND 关键字、逗号，或者给 SQL 语句前拼接 where、set 等后缀，可用于选择性插入、更新、删除或者条件查询等操作

```xml
<trim prefix="前缀" suffix="后缀" prefixOverrides="忽略前缀字符" suffixOverrides="忽略后缀字符">
    SQL语句
</trim>
```

##### bind

bind 标签可以通过 OGNL 表达式自定义一个上下文变量

```xml
<select id="selectWebsite" resultType="net.biancheng.po.Website">
    <bind name="pattern" value="'%'+_parameter+'%'" />
    SELECT id,name,url,age,country
    FROM website
    WHERE name like #{pattern}
</select>
```

##### 实现分页

使用 limit：

```xml
<select id="selectWebsite" resultType="net.biancheng.po.Website">
    SELECT id,name,url,age,country
    FROM website
    <trim prefix="where" prefixOverrides="and">
        <if test="site.name != null and site.name !=''">
            AND name LIKE CONCAT ('%',#{site.name},'%')
        </if>
        <if test="site.url!= null and site.url !=''">
            AND url LIKE CONCAT ('%',#{site.url},'%')
        </if>
        ORDER BY id limit #{from},#{pageSize}
    </trim>
</select>
```

### 逆向工程

- 导入依赖：

```xml
<dependency>
    <groupId>org.mybatis.generator</groupId>
    <artifactId>mybatis-generator-core</artifactId>
    <version>1.4.0</version>
</dependency>
```

- config 文件夹下创建 genertorConfig.xml 文件，用于配置及指定数据库及表等

```xml
<?xml version="1.0" encoding="UTF-8"?>
                    <!DOCTYPE generatorConfiguration PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN" "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
    <context id="DB2Tables" targetRuntime="MyBatis3">
        <commentGenerator>
            <!-- 是否去除自动生成的注释 -->
            <property name="suppressAllComments" value="true" />
        </commentGenerator>
        <!-- Mysql数据库连接的信息：驱动类、连接地址、用户名、密码 -->
        <jdbcConnection driverClass="com.mysql.jdbc.Driver"
                        connectionURL="jdbc:mysql://localhost:3306/test" userId="root"
                        password="root" />
        <!-- 默认为false，把JDBC DECIMAL 和NUMERIC类型解析为Integer，为true时 把JDBC DECIMAL 和NUMERIC类型解析为java.math.BigDecimal -->
        <javaTypeResolver>
            <property name="forceBigDecimals" value="false" />
        </javaTypeResolver>
        <!-- targetProject：生成POJO类的位置 -->
        <javaModelGenerator
                targetPackage="net.biancheng.pojo" targetProject=".\src">
            <!-- enableSubPackages:是否让schema作为包的后缀 -->
            <property name="enableSubPackages" value="false" />
            <!-- 从数据库返回的值被清理前后的空格 -->
            <property name="trimStrings" value="true" />
        </javaModelGenerator>
        <!-- targetProject：mapper映射文件生成的位置 -->
        <sqlMapGenerator targetPackage="net.biancheng.mapper"
                         targetProject=".\src">
            <!-- enableSubPackages:是否让schema作为包的后缀 -->
            <property name="enableSubPackages" value="false" />
        </sqlMapGenerator>
        <!-- targetProject：mapper接口生成的的位置 -->
        <javaClientGenerator type="XMLMAPPER"
                             targetPackage="net.biancheng.mapper" targetProject=".\src">
            <!-- enableSubPackages:是否让schema作为包的后缀 -->
            <property name="enableSubPackages" value="false" />
        </javaClientGenerator>
        <!-- 指定数据表 -->
        <table tableName="website"></table>
        <table tableName="student"></table>
        <table tableName="studentcard"></table>
        <table tableName="user"></table>
    </context>
</generatorConfiguration>
```

- 创建 GeneratorSqlmap 类执行生成代码

```java
public class GeneratorSqlmap {
    public void generator() throws Exception {
        List<String> warnings = new ArrayList<String>();
        boolean overwrite = true;
        // 指定配置文件
        File configFile = new File("./config/generatorConfig.xml");
        ConfigurationParser cp = new ConfigurationParser(warnings);
        Configuration config = cp.parseConfiguration(configFile);
        DefaultShellCallback callback = new DefaultShellCallback(overwrite);
        MyBatisGenerator myBatisGenerator = new MyBatisGenerator(config, callback, warnings);
        myBatisGenerator.generate(null);
    }
    // 执行main方法以生成代码
    public static void main(String[] args) {
        try {
            GeneratorSqlmap generatorSqlmap = new GeneratorSqlmap();
            generatorSqlmap.generator();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

