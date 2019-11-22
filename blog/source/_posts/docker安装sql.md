---
title: 使用docker安装MySQL
date: 2018-11-17 17:25:30
categories: "docker"
tags: ["docker", "SQL"]
---

## 使用docker安装MySQL

### 下载docker

``` bash
sudo apt-get install docker.io
```

### 更换源

``` bash
sudo vim /etc/docker/daemon.json{"registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]}
```

### 拉取镜像

``` sh
sudo docker pull mysql
```

### 创建容器

``` bash
sudo docker run --name mysql \-p 3306:3306 \-v /data/db/mysql:/var/lib/mysql \-e MYSQL_ROOT_PASSWORD=root \-d mysql
```

其中–name 是要创建容器的名字
-p为端口映射 `-p hostPort:containerPort`
-v表示给容器添加数据卷，这样数据就独立了，即使删除了容器也不会清除数据，`/data/db/mysql`是本地目录`/var/lib/mysql`是容器中MySQL默认的数据目录
`-e MYSQL_ROOT_PASSWORD=root`设置数据库root账户的默认密码

### 连接容器和数据库

``` bash
sudo docker exec -it mysql /bin/bashmysql -uroot -proot
```

## SQL Server

与MySQL类似，就直接贴代码了

```bash
docker run --name sql `-e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=testEmmm123!?" `-e "MSSQL_PID=Enterprise" -p 1433:1433 `-d "microsoft/mssql-server-linux"
```
