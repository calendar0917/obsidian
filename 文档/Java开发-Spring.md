---
title: "Spring"
subtitle: ""
description: "记录 Spring6 框架的学习"
date: 2025-10-10
lastmod: 2025-10-10
draft: false
toc:
 enable: true
weight: false
categories: ["开发"]
tags: ["技术"]
---

## 概述

### Spring FrameWork 特点

- 非侵入式
- 控制反转
- 面向切面编程
- 容器化管理
- 组件化
- 一站式

## IOC 控制反转

### 概述

Inversion of Control

- 用容器管理所有 Java 对象的实例化和初始化，控制依赖关系，称为 SpringBean

1. xml 配置文件
2. 抽象 `BeanDefinitionReader` 读取
3. 装配，读取信息，利用反射实例化
   - BeanFactory、ApplicationContex

4. 用 `Context.getBean("...")` 来获取

### 基于 xml 管理

#### 获取 Bean

xml 定义

```xml
<bean id="helloworldone" class="ccom.atguigu.spring6.bean.He11owor1d"></bean>
```

`Context.getBean("...")` 获取

```java
//根据类型获取接口对应bean
UserDaouserDao= context.getBean(UserDao.class);
```

#### 依赖注入

1. set 注入
   - xml 中进行配置
2. 构造器注入
   - ` <constructor-arggname="bnamevalue="java开发"></constructor-arg>`

> 是不是只能注入默认值？

3. 特殊值注入
   - 对象中注入其他对象（表示关系）
     - 法一：引入外部/内部类，bean 标签中嵌套 `ref`
     - 法二：级联赋值，直接嵌套对所注入的对象的属性的赋值

4. 数组类型注入
   - 配置中的 `<array>`标签

5. List 集合属性注入
   - 先定义 Bean，再用 `<list>` 标签注入

6. map 集合属性注入
   - `<map><entry><key>` 标签

7. 引用集合类型（？util 整合）

8. p 命名空间注入
   - 防止 7.中的util 的名称冲突

#### 引入外部属性

创建外部属性文件 `.property`

xml 中配置读取即可

#### 作用域

配置 scope 来指定作用域

- singleton 单例
- prototype 多例，每次获取时创建

#### 生命周期

![image-20251010211532962](https://raw.githubusercontent.com/calendar0917/images/master/image-20251010211532962.png)

#### FactoryBean 机制

根据接口实现 FactoryBean，自定义 getObjet（）方法返回值，来控制产生的对象

- 用于整合第三方框架(?)

### 基于注解注解管理

1. 开启组件扫描

```xml
<!--开启组件扫描功能-->
<context:component-scanbase-package="com.atguigu.spring6"></context:component-scan>
```

2. 注解说明

![image-20251010213939803](https://raw.githubusercontent.com/calendar0917/images/master/image-20251010213939803.png)

#### @Autowired 注入

1. set 方法注入

```java
@Autowired
public void setUserService(UserService userService) {
this.userService = userService;
}
```

2. 构造方法注入

3. 形参上注入

4. 根据名称进行注入（而非接口名）
   - 一个接口有多个实现类时 `@Qualifier`

#### @Resource 注入

默认根据 name 标签进行注入

### 全注解开发

无需使用配置文件，写一个配置类替代配置文件

```java
@configuration
//@componentscan({"com.atguigu.spring6.controller"
//"com.atguigu.spring6.service","com.atguigu.spring6.dao"})
@componentscan("com.atguigu.spring6")
public class Spring6config {}
```

### 手写 IoC

#### 反射

##### 获取对象：

```java
public void test01(）{
    //1 类名.cLass
    Class clazz1 = Car.class;
    //2 对象.getCLass()
    Class clazz2 =newv Car().getclass();
    //3 CLass.forName（"全路径"）
    Class clazz3 = Class.forName( className: "......");
    //实例化
    Object o = clazz3.getDeclaredConstructor().newInstance();
}
```

##### 获取方法：

```java
public void test02() throws Exception {
    Classclazz = Car.class;
    1/获取所有构造
    Constructor[]] constructors = clazz.getconstructors();
    for (Constructor c:constructors) {......}
    // 方法名：c.getname() 参数个数(public)：c.getConstructor()
    // 参数个数(所有)：c.getDeclaredConstructor()
}
```

##### 构造对象：

```java
Constructorc2=（clazz.getDeclaredConstructor(......)
c2.setAccessible(true);
Car car2 = (Car)c2.newInstance( ..initargs: "捷达"， 15,“白色");
```

##### 获取属性：

```java
Classclazz = Car.class;
//获取所有pubLic属性
Field[] fields = clazz.getFields();
//获取所有属性（包含私有属性）
Field[] fields = clazz.getDeclaredFields();
for (Field field:fields）{
System.out.println(field.getName());
}
```

##### 操作方法

```java
private方法
Method[] methodsAll = clazz.getDeclaredMethods();
for (Method m:methodsAll) {
//执行方法 runI
if(m.getName().equals("run")) {
    m.setAccessible(true);
    m.invoke(car) ;
}
```

#### 实现

##### 步骤

- 创建注解：@Bean创建对象、@Di属性注入

- 创建bean容器接口：ApplicationContext
- 定义方法，返回对象
- 实现bean容器接口：返回对象、根据包规则加载bean

##### 配置

@Bean

```java
@Target(ElementType.TYPE) // 目标
@Retention(RetentionPolicy.RUNTIME) // 运行范围
public @interface Bean
}
```

@Di

```java
```

ApplicationCotext 接口

```java
//创建有参数构造，传递包路径，设置包扫描规则
//扫描当前包及其子包，哪个类有@Bean注解，把这个类通过反射实例化
public AnnotationApplicationContext implement ApplicationContext(String basePackage) {
	// 路径转义,点替换为斜杠
    String packagePath = basePackage.replaceAll("\\.","\\\\")
    // 获取绝对路径
    Enumeration<URL> urls = Thread.currentThread().getContextClassLoader()·getResources(packagePath);
    while(urls.hasMoreElements()) {
        URL url = urls.nextElement();
        String filePath= URLDecoder.decode(url.getFile(), "utf-8");
        loadBean(new File(filePath)); // 实现 loadBean 方法
    }
    
    loadDi(); // 实现 loadDi 方法
}

	public static void loadBean(File file){
        // 1.判断当前内容是否是文件夹
        // 2.是，则获取当前文件夹所有内容      
        // 3.文件夹为空，返回空       
        // 4.文件夹不为空，遍历文件夹所有内容      
        // 4.1.遍历每个File对象，继续判断，如果还是文件，递归        
        // 4.2.不是文件夹，是文件        
        // 4.3.得到包文件 + 类名称部分    	
        // 4.4.判断当前文件类型是否.cLass        
        // 4.5.如果是.cLass类型，把路径\替换成。把.cLass去掉        
        // 4.6.判断类上面是否有注解@Bean，如果有实例化过程        
        // 4.7.把对象实例化之后，放到map集合bearFactory
        }
    private void loadDi() {
    	//实例化对象在beanFactory的map集合里面
    	//1遍历beanFactory的map集合
    	//2获取map集合每个对象（vaLue），每个对象属性获取到
    	//3遍历得到每个对象属性数组，得到每个属性
    	//4判断属性上面是否有@Di注解
    	//5如果有@Di注解，把对象进行设置（注入）
    }
```

## AOP 面向切面

### 引入

问题：

- 要抽取的代码在方法内部，无法通过抽取到父类解决

代理模式：

- 在调用目标方法时，不直接调用，而是通过代理类调用

- 代理：将非核心逻辑剥离出来以后，封装这些非核心逻辑的类、对象、方法
- 目标：代理“套用"了非核心逻辑代码的类、对象、方法

优化：

- 静态代理：再创建一个代理类，实现其他方法，再调用原有类
  - 问题：还是僵化，无法动态调整

- 动态代理：创建动态代理对象 ProxyFactory，用反射统一管理

**AOP**：通过预编译和动态代理，在不修改程序源码情况下，给程序添加功能

- 抽取横切关注点

- 整合横切关注点为通知方法
- 将各种通知方法整合为切面类

### 基于注解实现

分类：

- JDK：代理对象和目标对象实现相同接口（目标有接口时）
- cglib：通过继承目标类

- AspectJ：基于静态代理，将代理逻辑植入编译的字节码文件，效果是动态的，Spring借助了其中的注解

```java
@Aspect//切阻尖
@Component //ioc容器
public class LogAspect {
    //设置切入点和通知类型
    //通知类型：
    //前置 @Before(value = "切入点表达式")
    public void beforeMethod(JoinPoint joinPoint) {
        StringmethodName = joinPoint.getSignature().getName();
        Object[] args = jqinPoint.getArgs();
        // 获取连接点的信息
    }
    //返回 @AfterReturning
    //异常 @AfterThrowing
    //后置 @After()
    //环绕 @Around()
}
```

@Order 控制切面优先级

### 基于 xml 实现

```xml
<context:component-scanbase-package="com.atguigu.aop.xml"></context:component-scan>
<aop:config>
	<!--配置切面类-->
	<aop:aspect ref="loggerAspect">
	<aop:pointcut id="pointcut"
expression="execution(*com.atguigu.aop.xml.calculatorImpl.*(..))"/>
	<aop:before method="beforeMethod"pointcut-ref="pointcut"></aop:before>
	<aop:after method="afterMethod"pointcut-ref="pointcut"></aop:after>
	<aop:after-returning method="afterReturningMethod"returning="result"pointcut-
ref="pointcut"></aop:after-returning>
	<aop:after-throwing method="afterThrowingMethod"throwing="ex"pointcut-
ref="pointcut"></aop:after-throwing>
<aop:around method="aroundMethod"pointcut-ref="pointcut"></aop:around>
	</aop:aspect>
</aop:config>
```

## 事务

### JDBC Template

增：

```java
//1添加操作
//第一步编写sqL语句
String Sql = "INSERT INTO t_emp VALUES(NULL,?,?,?)";I
//第二步调用jdbcTempLate的方法，传入相关参数
//Object[］ params ={"东方不败"，20，"未知"};
//int rows = jdbcTemplate.update(sql,params);
int rows = jdbcTemplate.update(sql, ..args:"东方不败"，20，"未知"); 
```

改：

```java
//2修改操作
String sql ="update t_emp set name=? where id=?";
int rows =jdbcTemplate.update(sql, ..args: "林平之atguigu",3）;
```

删：

```java
//3删除操作
String sql'deletefromt_empwhere id=?";
introws=jdbcTemplate.update(sql, ..args: 3);
```

查：

```java
public void testSelectObject() {
    String sql="select*from t_emp
    List<Emp>list = jdbcTemplate.query(sql,
    new BeanPropertyRowMapper<>(Emp.class));
}
```

### 基于注解的声明式事务

保证事务的一致性、隔离性、持久性、原子性

`Transactional` 标签声明事务，可以设置：

- 只读
- 超时
- 回滚策略，哪些异常不回滚
- 隔离级别
- 传播行为，事务方法之间调用的处理逻辑

#### 全注解

不用 XML，改用配置类
