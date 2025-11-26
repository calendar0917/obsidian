---
title: "御林招新题：MySQL专题"
subtitle: "御林招新题：MySQL专题"
summary: "学习对数据库的基本操作"
description: "学习对数据库的基本操作"
image: ""
date: 2025-10-20
lastmod: 2025-10-20
draft: false
toc:
 enable: true
weight: false
hiddenFromHomePage: True
categories: ["CTF"]
tags: ["CTF"]
---

## **MySQL 安装与配置**

- **任务**：在你的 Linux 系统上安装 MySQL 服务器，并进行基本的安全配置。

- **具体操作**：

  - ~~使用系统包管理器安装 MySQL。~~

  > ```bash
  > # 下载 MySQL 5.7 的官方源，yum 默认不包含，不能直接下载
  > wget https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
  > # 安装源包，检查是否启用源
  > sudo rpm -ivh mysql57-community-release-el7-11.noarch.rpm
  > sudo yum repolist enabled | grep "mysql.*-community.*"
  > # 再安装
  > sudo yum install mysql-community-server -y
  > # 装到这里要确认密钥，有点麻烦，换方案，直接下载 tar 包
  > sudo systemctl start mysqld
  > # 安全脚本
  > sudo mysql_secure_installation
  > ```

  参考：[Linux 安装Mysql 详细教程（图文教程）_linux mysql安装教程-CSDN博客](https://blog.csdn.net/bai_shuang/article/details/122939884)

  下载 tar 包：[MySQL :: Download MySQL Community Server (Archived Versions)](https://downloads.mysql.com/archives/community/)，上传

  ```shell
  tar -zxvf mysql-5.7.35-linux-glibc2.12-x86_64.tar.gz
  # 创建目录并赋权
  groupadd mysql && useradd -r -g mysql mysql
  mkdir -p /data/mysql
  chown mysql:mysql -R /data/mysql
  chown mysql:mysql -R /usr/local/mysql
  chown mysql:mysql -R /tmp
  # 改配置
  vim /etc/my.cnf # 见下
  # 初始化
  cd /usr/local/mysql/bin/
  ./mysqld --defaults-file=/etc/my.cnf --basedir=/usr/local/mysql/ --datadir=/data/mysql/ --user=mysql --initialize
  # 启动
  cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysql
  service mysql start
  # 改密码操作比较繁琐，不做记录，参考文章
  ```

  ```cnf
  [mysqld]
  bind-address=0.0.0.0
  port=3306
  user=mysql
  basedir=/usr/local/mysql
  datadir=/data/mysql
  socket=/tmp/mysql.sock
  log-error=/data/mysql/mysql.err
  pid-file=/data/mysql/mysql.pid
  #character config
  character_set_server=utf8mb4
  symbolic-links=0
  explicit_defaults_for_timestamp=true
  ```

  - 运行 MySQL 的安全脚本，设置 root 密码，并删除不安全的用户和数据库。
  - **验证**：使用 `mysql -u root -p` 命令登录，确认能成功进入 MySQL 命令行。

  ```shell
  # 添加快速启动
  sudo ln -s /usr/local/mysql/bin/mysql /usr/bin/mysql
  # 注意：这样安装需要添加默认启动路径，如下
  # 登录
  [root@iZ2vc96n4f90pw7f8dfbfsZ bin]# mysql -u root -p
  Enter password:
  Welcome to the MySQL monitor.  Commands end with ; or \g.
  ```

```cnf
# /etc/systemd/system/mysql.service,配置service
[Unit]
Description=MySQL Server
After=network.target
[Service]
User=mysql
Group=mysql
ExecStart=/path/to/mysql-8.0.27/bin/mysqld --defaults-file=/path/to/mysql-8.0.27/my.cnf
ExecStop=/path/to/mysql-8.0.27/bin/mysqladmin --defaults-file=/path/to/mysql-8.0.27/my.cnf shutdown
Restart=on-failure [Install] WantedBy=multi-user.target
```

## **数据库操作与管理**

- **任务**：创建数据库、表，并进行数据的导入与导出。

- **具体操作**：

  - 在 MySQL 中创建一个新的数据库和一张表（例如，一个名为 `students` 的表，包含 `id`, `name`, `score` 等字段）。

  ```mysql
  # 建库
  CREATE DATABASE test_db;
  USE test_db;
  # 建表
  CREATE TABLE students (
      id INT AUTO_INCREMENT PRIMARY KEY,
      name VARCHAR(50) NOT NULL,
      score DECIMAL(5,2)
  );
  # 插入数据
  INSERT INTO students (name, score) VALUES 
  ('Alice', 85.5),
  ('Bob', 92.0),
  ('Charlie', 78.5);
  # 这里先出去 shell 导出
  # mysqldump -u root -p test_db > test_db_backup.sql
  # 删库
  DROP DATABASE test_db;
  # 重建
  CREATE DATABASE test_db;
  USE test_db;
  # 导入
  mysql -u root -p test_db < test_db_backup.sql
  ```

  ```shell
  mysqldump -u root -p test_db > test_db_backup.sql # 导出
  ```

  - 插入几条数据到表中。

  ![image-20251020191506632](https://raw.githubusercontent.com/calendar0917/images/master/image-20251020191506632.png)

  - **导出**：使用 `mysqldump` 命令将你的数据库导出为一个 SQL 文件。

  ```shell
  [root@iZ2vc96n4f90pw7f8dfbfsZ bin]# ./mysqldump -u root -p test_db > test_db_backup.sql
  Enter password:....
  [root@iZ2vc96n4f90pw7f8dfbfsZ bin]# ls | grep test
  ...
  test_db_backup.sql # 备份的表
  ```

  - **导入**：删除你创建的数据库，然后使用导出的 SQL 文件将其恢复。

  ```mysql
  
  mysql> SHOW DATABASES
      -> ;
  +--------------------+
  | Database           |
  +--------------------+
  | information_schema |
  | mysql              |
  | performance_schema |
  | sys                |
  | test_db            |
  +--------------------+
  5 rows in set (0.00 sec)
  ```

  

## **数据库性能调优**

- **任务**：了解并修改 MySQL 的关键配置参数，以提高性能。

- **具体操作**：

  - 找到 MySQL 的配置文件（通常是 `/etc/mysql/my.cnf` 或 `/etc/my.cnf`）。

  - **挑战**：

    - 修改 `innodb_buffer_pool_size` 参数，并简要解释该参数的作用。

    > InnoDB 存储引擎极为关键的参数，它**指定了 InnoDB 缓冲池的大小**。InnoDB 缓冲池主要用于缓存表数据、索引数据等，当数据库进行查询操作时，会**先从缓冲池中查找所需数据**，如果能找到（即命中缓存），就可以避免从磁盘读取数据，从而极大地**提高查询性能**。

    - 修改 `max_connections` 参数，并解释其对系统资源和并发连接的影响。

    > 用于设置 MySQL 服务器允许的**最大并发连接数**。过小会导致部分连接被拒绝，过多会占用服务器资源。

    ```cnf
    [root@iZ2vc96n4f90pw7f8dfbfsZ local]# vi /etc/my.cnf
    [mysqld]
    bind-address=0.0.0.0
    port=3306
    ......
    innodb_buffer_pool_size = 2G
    max_connections = 500
    ```

  - **验证**：重启 MySQL 服务，并确认新的配置已生效。

  ```shell
  [root@iZ2vc96n4f90pw7f8dfbfsZ support-files]# mysql -u root -p
  ......
  
  mysql> SHOW VARIABLES LIKE 'innodb_buffer_pool_size';
  +-------------------------+------------+
  | Variable_name           | Value      |
  +-------------------------+------------+
  | innodb_buffer_pool_size | 2147483648 |
  +-------------------------+------------+
  1 row in set (0.01 sec)
  
  mysql> SHOW VARIABLES LIKE 'max_connections';
  +-----------------+-------+
  | Variable_name   | Value |
  +-----------------+-------+
  | max_connections | 500   |
  +-----------------+-------+
  1 row in set (0.00 sec)
  ```

## **应用程序集成与数据处理**

- **任务**：编写一个简单的应用程序，连接到你的 MySQL 数据库，执行查询和计算，并将结果导出。

- **具体操作**：

  - **语言选择**：你可以使用任何你熟悉的语言（**推荐使用Python**）。
  - **程序功能**：
    - 连接到你在必做部分创建的数据库。
    - 查询 `students` 表中的所有数据。
    - 计算学生的平均分数。
    - 将所有数据（包括计算出的平均分）写入一个名为 `report.csv` 的 CSV 文件中。
  - **要求**：在代码中添加注释，解释连接数据库、执行查询和写入文件的关键步骤。

  ```python
  # 代码模板
  import pymysql
  import csv
  
  # 数据库连接配置
  db_config = {
      "host": "8.137.38.223",  # MySQL 主机地址
      "user": "root",  # 数据库用户名
      "password": "1234",  # 数据库密码
      "database": "test_db"  # 数据库名
  }
  
  # 连接数据库
  conn = pymysql.connect(**db_config)
  cursor = conn.cursor()
  
  # 查询 students 表中的所有数据
  query_sql = "SELECT * FROM students"
  cursor.execute(query_sql)
  students_data = cursor.fetchall()
  
  # 获取表的列名
  column_names = [desc[0] for desc in cursor.description]
  
  # 计算学生的平均分数
  scores = [row[2] for row in students_data]  # 假设分数在第三列（索引为2）
  average_score = sum(scores) / len(scores) if scores else 0
  
  # 将所有数据写入 report.csv 文件
  with open("report.csv", "w", newline="") as csvfile:
      writer = csv.writer(csvfile)
      # 写入列名
      writer.writerow(column_names + ["average_score"])
      # 写入每行数据以及平均分
      for row in students_data:
          writer.writerow(list(row) + [average_score])
  
  # 关闭游标和连接
  cursor.close()
  conn.close()
  
  print("数据查询、计算及导出完成，结果已保存至 report.csv 文件")
  ```

> - pymysql 库的使用：
>
>   - `conn = pymysql.connect()` 建立连接，传入 `host port user password database`，连接成功，返回一个 conn 对象，用这个对象操作
>
>   - `cursor = conn.cursor()` 创建游标对象，用于执行 sql 语句
>
>     - 还有`conn.cursor(pymysql.cursors.DictCursor)`，返回一个字典游标，用于插入字典
>
>   - `cursor.execute("...")`，执行
>
>     - 防注入写法：
>
>     - ```python
>       data = ("Alice", 85.5)
>       sql = "INSERT INTO students (name, score) VALUES (%s, %s)"
>       cursor.execute(sql, data)
>       ```
>
>   - 获取数据：`fetchall() fetchone() fetchmany(size)`
>
>   - 释放资源：`cursor.close() conn.close()`

![image-20251020202548553](https://raw.githubusercontent.com/calendar0917/images/master/image-20251020202548553.png)

## **自动化备份与恢复**

- **任务**：编写一个 Shell 脚本，自动化数据库的日常备份。
- **具体操作**：
  - 编写一个 Shell 脚本，使用 `mysqldump` 命令备份你的数据库。
  - 脚本应为备份文件自动添加时间戳，例如 `backup_db_2025-09-15.sql`。
  - 使用 `crontab` 将该脚本设置为每天凌晨自动运行一次。
  - **验证**：手动执行脚本，并检查是否成功生成了带有时间戳的备份文件。

```sh
#!/bin/bash

DB_USER="root"           # 用户名
DB_PASSWORD="...."  # MySQL 密码
DB_NAME="test_db"        # 要备份的数据库名
BACKUP_DIR="/usr/local/mysql/backup"  # 备份文件保存目录
DATE=$(date +"%Y-%m-%d")
# 命令行中，date +%Y-%m-%d 可以格式化 date 输出
BACKUP_FILE="$BACKUP_DIR/backup_${DB_NAME}_${DATE}.sql"  # 备份文件名
# 使用 mysqldump 备份数据库
# 这里得要指定路径，系统变量没配置好
/usr/local/mysql/bin/mysqldump -u ${DB_USER} --password=${DB_PASSWORD} ${DB_NAME} > ${BACKUP_FILE}

# 检查备份是否成功
if [ $? -eq 0 ]; then  # $? 是特殊变量，返回上一条语句的执行结果，成功返回0
    echo "备份成功！文件：${BACKUP_FILE}"
else
    echo "备份失败！"
fi 
```

```shell
# 赋权
chmod +x mysql_backup.sh
# 手动执行
./mysql_backup.sh
备份成功！文件：/usr/local/mysql/backup/backup_test_db_2025-10-20.sql
# 脚本定时执行，用 crontab 定时工具
crontab -e
# 加上：
0 1 * * * /usr/local/mysql/mysql_backup.sh
```

![image-20251020204004953](https://raw.githubusercontent.com/calendar0917/images/master/image-20251020204004953.png)

- 不知道怎么关闭 mysql

> 源码编译，需要到指定目录：
>
> ```shell
> [root@iZ2vc96n4f90pw7f8dfbfsZ mysql]# cd /usr/local/mysql/bin
> [root@iZ2vc96n4f90pw7f8dfbfsZ bin]# sudo ./mysqladmin -u root -p shutdown
> ```

- 如何看系统占用？

  - top 指令

  - 增强版 htop