---
title: mysql工作笔记
category: MySQL技术
date: 2020-04-02 20:47:28
tags: MySQL技术
---

<!-- more -->


> mysql分配用户以及相应的权限,设置远程访问
	
```sql
# 1.新建用户
CREATE USER username IDENTIFIED BY 'password';

# 2.分配用户权限
GRANT privileges ON databasename.tablename TO userName (WITH GRANT OPTION);
如果添加WITH GRANT OPTION代表该用户还可以为其他用户授权，否则没有为其他用户授权的权限

# 3.允许远程访问
## 1）更改表
update mysql.user set `Host` = "%" where `User`="userName" and `Host`="localhost"
## 2)直接授权
GRANT ALL PRIVILEGES ON databasename.tablename TO 'userName'@'%|ipaddress' IDENTIFY BY 'password' WITH GRANT OPTION
## [%表示任意主机，ipaddress表示指定的ip可以访问]

## 最后要更改mysql的配置
sudo vim /etc/mysql/my.cnf
bind-address:127.0.0.1 注释掉

## 然后重启mysql
sudo service mysql restart

# 4.最后如果有修改权限，要刷新权限
FLUSH PRIVILEGES;
```
	
> 修改密码

```sql
# mysql>
SET PASSWORD FOR 'root'@'localhost' = PASSWORD('newpass');

# mysqladmin
mysqladmin -u root password "newpass"
mysqladmin -u root password oldpass "newpass"  # root已经设置过密码

# 用UPDATE直接编辑user表
　mysql> use mysql;
　mysql> UPDATE user SET Password = PASSWORD('newpass') WHERE user = 'root';
　mysql> FLUSH PRIVILEGES;
```

> 导入导出数据库

```sql
## 导入sql文件
1. 连接进入mysql数据库
mysql -h192.168.229.172 -ucc_test -pcc_test -P3306 cctestaudit
source path/sql.sql

2. 直接命令导入
mysql -h192.168.229.172 -ucc_test -pcc_test -P3306 cctestaudit < path/sql.sql

3. 导出整个数据库中的所有数据
mysqldump -h192.168.229.172 -ucc_test -pcc_test -P3306 cctestaudit > fileName.sql

4. 导出数据库中某张表
mysqldump -h192.168.229.172 -ucc_test -pcc_test -P3306 cctestaudit tableName > fileName.sql

5. 导出数据库中所有表结构
mysqldump -h192.168.229.172 -ucc_test -pcc_test -P3306 -d cctestaudit > fileName.sql

6. 导出某张表的结构
mysqldump -h192.168.229.172 -ucc_test -pcc_test -P3306 -d cctestaudit tableName > fileName.sql
```

> 创建数据库指定编码
```text
CREATE DATABASE `mydb` CHARACTER SET utf8 COLLATE utf8_general_ci;
```