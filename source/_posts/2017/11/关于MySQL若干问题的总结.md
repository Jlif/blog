---
layout: "post"
title: "关于 MySQL 若干问题的总结"
date: "2017-11-08"
categories: 数据库
---

今天碰到一个问题，就是往数据库存中文名字的时候，发现存不进去，错误提示如下：
```mysql
[HY000][1366] Incorrect string value
```
简单搜索了一下，发现可能是编码设置的问题，在 MySQL 中输入 `show variables like 'character%';` 命令，就可以看到自己的 MySQL 数据库相关的设置了，MySQL 默认的设置是 `Latin1`。我按照网上的相关说明，拷贝了一个 `my.cnf` 设置文件进去，设置好相关的编码，然后把数据表全删了，重新生成，发现还是插入不了中文数据，我当时有点奔溃。
<!-- more -->
这个时候，我就想了，是不是因为自己的 MySQL 版本太低了。因为用的是阿里云的 centOS 6.9 的系统，MySQL 是用 yum 安装的，版本是 5.1.73，算是比较低的了。

#### 升级 MySQL

于是，我去 MySQL 官网下载了一个 rpm 包，就是 mysql 的 repo，上传到服务器上，安装了之后一更新，把 MySQL 就升级到 5.7 了，但是这个时候我发现 MySQL 启动不了了。

#### 初始化 MySQL

使用 `less /var/log/mysql.log` 命令查看 MySQL 的日志，发现说的大概意思好像是 ibdata1 分页大小什么的，在网上搜了下，按照相关步骤干脆把 `/var/lib/mysql` 里的文件全删除了。（注意：这里面有数据库文件，删除需谨慎）

然后执行 `mysqld --initialize --user=mysql` 初始化 MySQL。

这个时候执行 `service mysqld start`，发现能启动成功，但是登录的时候，发现原来的密码什么的不管用了，这个时候，应该安全启动 MySQL，就是不校验用户的那种。

先执行 `service mysqld stop` 把 MySQL 给关闭掉，然后安全启动 MySQL，有以下两种方式：
1. 在 `my.cnf` 文件中 [mysqld] 下面添加一行配置，`skip-grant-tables`，就是跳过鉴权，然后执行 `service mysqld start` 启动 MySQL
2. 直接执行 `mysqld_safe --user=mysql --skip-grant-tables --skip-networking &`

使用上述两种方法都能够启动 MySQL，不需要密码就能使用 MySQL

#### 重置密码

- 输入 `mysql` 命令，启动 MySQL

- 执行 `mysql> update user set authentication_string=PASSWORD('root') where user='root';`，重新设置密码

- 开启远程访问：`mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%'IDENTIFIED BY 'jiangchen' WITH GRANT OPTION;`

- 再执行`mysql> FLUSH PRIVILEGES;`

- 这个时候退出数据库，正常启动，重新登陆之后，会发现提示修改密码，于是我们修改密码

```mysql
mysql> show databases;
ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.
mysql> set PASSWORD=Password('root');
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

再执行如下命令
```mysql
mysql> alter user root@localhost PASSWORD EXPIRE NEVER;
Query OK, 0 rows affected (0.00 sec)

mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.01 sec)
```
这样密码就修改成功了，密码也不会提示过期了。

#### 修复 MySQL 不能存中文的问题

上面一通折腾，只是重置了 MySQL 的密码，但是我存中文还是存不了，还得整 utf8 的问题。

于是我把编码按照之前做的又都设置为 utf8，具体操作步骤为在 `my.cnf` 配置文件中添加一行：`character-set-server=utf8`

这样做了之后发现还是不行，删了表重新建也不行。

最后，我把 database 删除了，重新建了一个 database，然后生成表，发现这个时候就可以存中文了。

#### 总结

> 其实我没有必要升级数据库的，一开始设置好编码，删除库，重新建应该就好了。但是在这个过程中，也折腾了不少东西，学到了不少东西吧。数据库在创建的时候编码方式就设置好了，针对单张表除非特别声明，使用自定义的编码方式，不然都是默认的，尤其是我用的还是 jpa 的自动生成表。