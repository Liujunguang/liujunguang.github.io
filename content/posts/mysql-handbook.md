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

