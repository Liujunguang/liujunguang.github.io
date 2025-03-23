+++
title = 'Mysql Handbook'
date = 2024-05-04T23:38:20+08:00
draft = false
+++

## 简介

MySQL是目前应用最广泛的开源关系数据库。MySQL最早是由瑞典的MySQL AB公司开发，该公司在2008年被SUN公司收购，
紧接着，SUN公司在2009年被Oracle公司收购，所以MySQL最终就变成了Oracle旗下的产品。

## 部署

### 1. docker 容器部署

```shell
#!/bin/bash

# mysql root password
pwd=123456

sudo docker run --name mysql -p 3306:3306 \
	-v /data/mysql/logs:/var/log/mysql \
	-v /data/mysql/data:/var/lib/mysql \
	-v /data/mysql/mysql-files:/var/lib/mysql-files \
	-e MYSQL_ROOT_PASSWORD=${pwd} \
	-d mysql:8.4.0
```

### 2. 禁止外部 root 权限访问

刚创建的 mysql 数据库，外部也可以通过 root 用户名密码访问，通过下面操作关闭：

```shell script
# 登陆 mysql cli
docker exec -it mysql bash

mysql -u root -p
# 输入 root 密码

# 切换 mysql 库
use mysql;
# 查看当前允许登陆的用户
mysql> select host, user from user;
+-----------+------------------+
| host      | user             |
+-----------+------------------+
| %         | root             |
| localhost | mysql.infoschema |
| localhost | mysql.session    |
| localhost | mysql.sys        |
| localhost | root             |
+-----------+------------------+
5 rows in set (0.00 sec)
```

说明：如果 `host` 为 '%'，则说明所有 IP 都可以 root 权限访问 mysql，修改只有特定 ip 可以远程 root 访问 mysql：

```shell script
# xxxx 是对应的 ip，如果只希望本机访问，可以用 localhost
# update user set host='xxxx' where user='root';

# 这里已经有 `localhost` 的 root 访问权限，我们希望删除 root 的外部访问权限
delete from user where host='%' and user='root';

# 创建新的用户外部访问权限，并设置密码
create user 'username'@'%' identified by 'password';

# 为新创建的用户，授予数据库操作权限，`*.*` 代表所有数据表的所有权限
grant all privileges on *.* to 'username'@'%';

# 刷新，使更新的配置生效
flush privileges;
```

### 3. 通过命令行建表，无法使用中文

通过 `docker exec` 的方式，命令行建表时，无法复制粘贴中文，设置环境变量即可：
```shell script
docker exec -it mysql env LANG=C.UTF-8 bash
```

## 概念

### 主键（primary key）

一列（或一组列），能够唯一区分表中每个行。

> `应该总是定义主键`：虽然允许不定义主键，但大多数数据库设计人员都应该保证创建的每个表都具有一个主键，以便后续都数据库操作和管理。

表中任何列都可以作为主键，只要满足以下条件：

- 任意行的值唯一
- 不为 NULL

主键通常定义在一列上，但也允许使用多个列作为主键，此时所有列的组合，必须是唯一的（单个列的值可以重复）

### 通配符

`mysql 中命令不区分大小写`

使用通配符（*），返回表中所有列，返回列顺序，一般是在表定义时的顺序。
```shell script
SELECT * FROM products;
```

> 一般除非确实需要表中的每个列，否则最好别使用 * 通配符，因为检索不需要的列通常会降低检索和应用程序的性能。

### DISTINCT

有时搜索出的表中数据可能重复：

```shell script
SELECT vend_id FROM products;
```

![](/images/mysql/distinct-1.jpeg)

使用 DISTINCT 限定只返回不同的值：

```shell script
SELECT DISTINCT vend_id FROM products;
```

![](/images/mysql/distinct-2.jpeg)

### LIMIT

单个参数，代表限制不超过该行数。

```shell script
# 返回 5 行
SELECT prod_name FROM products LIMIT 5;
```

两个参数，第一个数为起始位置，第二个为要返回的行数：

```shell script
# 返回行5（第6行）开始，一共返回5行
SELECT prod_name FROM products LIMIT 5,5;
```

> 行0：表示第一行
>行数不够时，MySQL 只返回它能返回的行数。

### 使用数据处理函数

#### 常用文本处理函数

| 函数          | 说明           |
|-------------|--------------|
| Left()      | 返回串左边的字符     |
| Length()    | 返回串的长度       |
| Locate()    | 找出串的一个子串     |
| Lower()     | 将串转换为小写      |
| LTrim()     | 去掉串左边的空格     |
| Right()     | 返回串右边的字符     |
| RTrim()     | 去掉串右边的空格     |
| Soundex()   | 返回串的SOUNDEX值 |
| SubString() | 返回子串的字符      |
| Upper()     | 将串转换为大写      |

> `SOUNDEX` 是一个将任何文本串转换为描述其语音表示的字母数字模式的算法。它匹配所有发音类似于该字符串的结果。

#### 日期和时间处理函数

| 函数            | 说明               |
|---------------|------------------|
| AddDate()     | 增加一个日期（天、周等）     |
| AddTime()     | 增加一个时间（时、分等）     |
| CurDate()     | 返回当前日期           |
| CurTime()     | 返回当前时间           |
| Date()        | 返回日期时间的日期部分      |
| DateDiff()    | 计算两个日期之差         |
| Date_Add()    | 高度灵活的日期运算函数      |
| Date_Format() | 返回一个格式化的日期或时间字符串 |
| Day()         | 返回一个日期的天数部分      |
| DayOfWeek()   | 对于一个日期，返回对应的星期几  |
| Hour()        | 返回一个时间的小时部分      |
| Minute()      | 返回一个时间的分钟部分      |
| Month()       | 返回一个日期的月份部分      |
| Now()         | 返回当前日期和时间        |
| Second()      | 返回一个时间的秒部分       |
| Time()        | 返回一个日期时间的时间部分    |
| Year()        | 返回一个日期的年份部分      |

#### 数值处理函数

| 函数     | 说明        |
|--------|-----------|
| Abs()  | 返回一个数的绝对值 |
| Cos()  | 返回一个角度的余弦 |
| Exp()  | 返回一个数的指数值 |
| Mod()  | 返回除操作的余数  |
| Pi()   | 返回圆周率Pi   |
| Rand() | 返回一个随机数   |
| Sin()  | 返回一个角度的正弦 |
| Sqrt() | 返回一个数的平方根 |
| Tan()  | 返回一个角度的正切 |

#### 聚合函数（aggregate function）

| 函数      | 说明       |
|---------|----------|
| AVG()   | 返回某列的平均值 |
| COUNT() | 返回某列的行数  |
| MAX()   | 返回某列的最大值 |
| MIN()   | 返回某列的最小值 |
| SUM()   | 返回某列值之合  |

### 分组

#### 分组 `GROUP BY`

```shell script
SELECT vend_id, COUNT(*) AS num_prods FROM products GROUP BY vend_id;
```

+---------+-----------+
| vend_id | num_prods |
+---------+-----------+
|    1001 |         3 |
|    1002 |         2 |
|    1003 |         7 |
|    1005 |         2 |
+---------+-----------+

#### 过滤分组 `HAVING`

HAVING 非常类似于 WHERE，所有类型的 WHERE 子句都可以用 HAVING 来替代。唯一的差别就是 WHERE 过滤行，而 `HAVING` 过滤分组。

<br/>
也可以理解成 `WHERE 在数据分组前进行过滤，HAVING 在分组之后进行过滤`。

```shell script
SELECT cust_id, COUNT(*) AS orders FROM orders GROUP BY cust_id HAVING COUNT(*) >= 2;
```

+---------+--------+
| cust_id | orders |
+---------+--------+
|   10001 |      2 |
+---------+--------+

列出具有2个（含）以上、价格为10（含）以上的产品的供应商。

```shell script
SELECT vend_id, COUNT(*) AS num_prods
FROM products
WHERE prod_price >= 10
GROUP BY vend_id
HAVING COUNT(*) >= 2;
```

+---------+-----------+
| vend_id | num_prods |
+---------+-----------+
|    1003 |         4 |
|    1005 |         2 |
+---------+-----------+

<font color='red'>GROUP BY 的输出可能不是分组的顺序，所以如果要保证输出顺序，一般在使用 GROUP BY 子句时，也应该给出 ORDER BY，这是保证数据正确的唯一方法。</font>

## 联结表

![](/images/mysql/join.png)

## 索引

在 **数据库管理系统**（DBMS）中，索引是提高数据检索速度的重要工具。通过使用索引，数据库可以快速找到所需的数据，而不必扫描整个表。
索引属于 `存储引擎` 级别的概念，不同的存储引擎对索引的实现方式不同。

### 索引类型

MySQL 中索引类型包括：
- B+ 树索引（默认）
- Hash 索引
- 全文索引
- 空间索引

其中 B+ 树索引大致可分为两类：
- 聚簇索引
- 非聚簇索引

### 聚簇索引（Clustered Index）

#### 定义

一种数据存储方式，索引中键值的逻辑顺序与数据行的物理顺序相同，一个表只能有一个聚簇索引。
（可以类比新华字典的拼音目录，A-X的顺序目录，实际字也是按 A-X 的顺序保存内容）

#### 特性

- 聚簇索引叶子节点保存的是行数据
- 检索效率更高，范围查询、排序操作效率高
- 如果主键可更改，由于数据按索引键排序，插入操作可能导致数据页的重新排列，性能开销大
- 通常在主键上创建聚簇索引，因为主键的唯一性和非空特性适合
- 一个表只能有一个聚簇索引

#### 创建聚簇索引

```sql
CREATE TABLE Employees (
    EmployeeID INT PRIMARY KEY,
    FirstName VARCHAR(50),
    LastName VARCHAR(50),
    BirthDate DATE,
    HireDate DATE
);

-- 在 EmployeeID 列上创建聚集索引
CREATE CLUSTERED INDEX IX_Employees_EmployeeID ON Employees (EmployeeID);
```

### 非聚簇索引（Non-clustered Index 或 Secondary Index）

#### 定义

索引的逻辑顺序，和磁盘上的物理存储顺序不同，一个表中可以拥有多个非聚簇索引。
（可以类比新华字典的偏旁部首目录，实际字并不是按偏旁部首顺序存放）

#### 特性

- 辅助索引访问数据总是需要二次查找，查找效率相对较低
- 辅助索引叶子节点保存的是主键值
- 插入操作性能开销低，不影响数据表的物理存储顺序
- 需要额外的存储空间来保存索引页
- 一个表中可以拥有多个非聚簇索引

#### 创建非聚簇索引

```sql
CREATE TABLE Employees (
    EmployeeID INT PRIMARY KEY,
    FirstName VARCHAR(50),
    LastName VARCHAR(50),
    BirthDate DATE,
    HireDate DATE
);

-- 在 LastName 列上创建非聚集索引
CREATE NONCLUSTERED INDEX IX_Employees_LastName ON Employees (LastName);
```

### 索引（Index）和键（Key）的区别

在实际使用中，MySQL 的 Key 和 Index 经常被混用

#### Key 

- 数据库的 `逻辑结构`
- 如主键（PRIMARY KEY）、唯一键（UNIQUE KEY）、外键（FOREIGN KEY）
- 包含 `约束` + `索引`
- MySQL 在创建约束（如主键、唯一键、外键）时，会自动为其建立索引
- 约束的作用时维护数据的完整性、一致性，而索引的作用时提高查询效率，索引本身并不提供约束字段的行为

#### Index

- 数据库的 `物理结构`
- 辅助查询，提高检索速度
- 通常存储在 InnoDB 表空间中，以目录结构形式存在

## 全文本搜索

> 注意: `InnoDB` 不支持全文本搜索。

// TODO: 用到具体的再来补充

