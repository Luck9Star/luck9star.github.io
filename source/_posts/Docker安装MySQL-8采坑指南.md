---
title: Docker安装MySQL 8采坑指南
tags:
  - Docker
abbrlink: 28185
date: 2018-06-10 17:50:00
categories:
---
## 1.Docker 方式安装
`docker run --name mysql -v /some-mysql-file-path:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw -e MYSQL_DATABASE=some-created-db -p 3306:3306 -d mysql:8`
## 2.Docker Compose 方式安装
```
version: '3'
services:
  mysql:
    image: mysql:8
    volumes:
       - /some-mysql-file-path:/var/lib/mysql
    networks:
       mysql:
         aliases:
           - db
    restart: unless-stopped
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=my-secret-pw
      - MYSQL_DATABASE=some-created-db
    #env_file: mysql.env
    sysctls:
      net.core.somaxconn: 1024
      net.ipv4.tcp_syncookies: 0
    ulimits:
      nproc: 65535
      nofile:
        soft: 20000
        hard: 40000
networks:
  mysql:
```
> 如果使用env_file配置的话:
```
# mysql.env
MYSQL_ROOT_PASSWORD=881007yyt
MYSQL_DATABASE=common_db
```
`dockr-compose up -d`

## 3.参数说明
MySQL将会把其数据库文件安装在本地`/some-mysql-file-path`路径下.

如果以后需要更新或者想要对MySQL做一些操作的话可以直接继续mount这个路径.

命令行`-e` docker-compose的environment后面加的是一些环境变量
- `MYSQL_ROOT_PASSWORD`是数据库root用户的初始密码.**这个环境变量是必须的.**
- `MYSQL_DATABASE`是运行该镜像后新建的数据库,新建数据库只会在创建数据库文件时候发生,即完整新建MySQL数据库时.
- `MYSQL_USER`, `MYSQL_PASSWORD`是运行该镜像后新建的数据库用户.
- `MYSQL_RANDOM_ROOT_PASSWORD`表示root用户密码将会随机生成,生成的密码将会咋日志中打印出来.
> docker logs ...


## 4. 其他
- 日志报错:`mbind: Operation not permitted`
> MySQL运行时会报该错误,这是MySQL docker的一个bug,原因大致是MySQL期望获取到独占的内存,目前看来不影响使用.
- MySQL可以直接通过IP:3306端口在任意位置访问.
- 其他容器需要访问MySQL的话需要连接MySQL网络`docker network connect --alias container mysql_mysql container` 这是可以通过db:3306访问MySQL了

## 5.MySQL8的问题
MySQL8.0用到了新的密码插件验证方式，5.7叫做mysql_native_password，8.0叫做caching_sha2_password，这种加密方式让很多和MySQL连接的界面工具(如Navicat)或编程语言(如PHP)mysqli接口失效:
> Error : The server requested authentication method unknown to the client [caching_sha2_password]

这时需要进入MySQL修改密码:
```
docker exec -it mysql_mysql_1 sh -c 'exec mysql -uroot -p"$MYSQL_ROOT_PASSWORD"'
```
进入MySQL命令行后:
```
use mysql;
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '你的密码';
FLUSH PRIVILEGES;
```

## 6.备份数据库

```
docker exec mysql_mysql_1 sh -c 'exec mysqldump --all-databases -uroot -p"$MYSQL_ROOT_PASSWORD"' > /some/path/on/your/host/all-databases.sql
```
数据将会保存到```/some/path/on/your/host/all_databases_`+%y_%m_%d_%H_%m`.sql```

