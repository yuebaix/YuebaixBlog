+++
title       = "七天构建SpringCloud集群"
subtitle    = "day2：基础设施搭建"
description = "用docker-compose完成中间件搭建"
author      = "月白"
date        = 2021-10-02T19:00:00+08:00
URL         = ""
image       = ""
categories  = ["TECH"]
tags        = ["干货", "教程"]
+++

## 一、安装 Docker Desktop

创建桥接子网

> docker network create -d bridge --subnet=172.18.0.0/16 --gateway 172.18.0.1 inner-network

## 二、配置文件

```yaml
version: '3.1'

services:

  mysql:
    image: mysql:5.7.35
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW --character-set-server=utf8mb4 --collation-server=utf8mb4_general_ci
    restart: always
    container_name: mysql
    dns:
      - 119.29.29.29
      - 114.114.114.114
      - 8.8.8.8
    networks:
      default:
        ipv4_address: 172.18.0.2
    ports:
      - "13306:3306"
    volumes:
      - E:\jiuzhou\volumes\mysql\config:/etc/mysql/conf.d
      - E:\jiuzhou\volumes\mysql\data:/var/lib/mysql
      - E:\jiuzhou\volumes\mysql\initsql:/docker-entrypoint-initdb.d
    environment:
#      MYSQL_ROOT_PASSWORD: 'yuebaix'
      MYSQL_ALLOW_EMPTY_PASSWORD: 'yes'
      TZ: Asia/Shanghai

  zk:
    image: zookeeper:3.7.0
    restart: always
    container_name: zk
    hostname: zk
    dns:
      - 119.29.29.29
      - 114.114.114.114
      - 8.8.8.8
    networks:
      default:
        ipv4_address: 172.18.0.11
    ports:
      - "12181:2181"
#      - "11011:8080"
    volumes:
      - E:\jiuzhou\volumes\zk\data:/data
      - E:\jiuzhou\volumes\zk\datalog:/datalog
      - E:\jiuzhou\volumes\zk\logs:/logs
    environment:
      ZOO_STANDALONE_ENABLED: 'true'
      ZOO_ADMINSERVER_ENABLED: 'false'

  apollo-quick-start:
    image: yuebaix/apollo-quick-start
    container_name: apollo-quick-start
    dns:
      - 119.29.29.29
      - 114.114.114.114
      - 8.8.8.8
    networks:
      default:
        ipv4_address: 172.18.0.21
    ports:
      - "11021:8080"
      - "11022:8090"
      - "11023:8070"
    environment:
      JAVA_OPTS: '-Xms100m -Xmx1000m -Xmn100m -Xss256k -XX:MetaspaceSize=10m -XX:MaxMetaspaceSize=250m'
    depends_on:
      - mysql
    links:
      - mysql:apollo-db

  xxl:
    image: xuxueli/xxl-job-admin:2.3.0
    restart: always
    container_name: xxl
    dns:
      - 119.29.29.29
      - 114.114.114.114
      - 8.8.8.8
    networks:
      default:
        ipv4_address: 172.18.0.31
    ports:
      - "11031:8080"
    volumes:
      - E:\jiuzhou\volumes\xxl\applogs:/data/applogs
    environment:
      PARAMS: |
        --spring.datasource.url=jdbc:mysql://mysql:3306/xxl_job?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai
        --spring.datasource.username=root
        --spring.datasource.password=
    depends_on:
      - mysql
    links:
      - mysql

networks:
  default:
    external:
      name: inner-network
```

## 三、常用命令

```shell
docker build -t image-name .
docker-compose -f docker-compose.yml up

docker tag origin-image-name tag-image-name
docker push tag-image-name

docker pull tag-image-name

docker run -it image-name /bin/bash
docker exec -it container-name /bin/bash
docker cp container-name:/path .
```

## 四、比较折腾

### 1.C盘爆了

> 不经常用家里的电脑进行开发，而且本身C盘容量已经很小了，只剩6个G，一边开发一边监控硬盘大小，最终将gradle跟maven的本地缓存配置到其他盘，暂时缓解这个问题。

### 2.C盘又爆了

> 准备用docker搭建集群，结果因为docker desktop升级了wsl2，配置本地路径的配置没了，本地镜像与运行时文件，都挤在C盘，最终决定把后面一个盘的空间
> 释放出来，合并到C盘，中间备份分类文件，安装启动U盘，各种费力。

### 3.内存爆了

> 最后C盘扩充了80个G，应该够用很长时间了。在叠加中间件的时候发现win的linux虚拟机内存占用很高，加上idea一下子什么都不干4个G没了，操作系统启动也得2个G。
> apollo耗费内存接近2个G，于是决定优化一下JVM启动参数。官方快速启动的镜像不支持环境变量调整启动参数，决定拉下代码来，自己改一把。

### 4.被换行符坑了

> 本来很简单的一个功能，感觉做起来也不复杂。结果就是各种运行报错，最终定位到是换行符的问题。idea导入文件按照操作系统直接给改成CRLF了，改完配置为LF，
> 重新run一遍通过。这事很蛋疼，一个是因为idea自动按操作系统给你转很蛋疼，一个是alpine里换行符会造成shell执行不下去，果然开发只能用mac。改完了顺道
> 给宋顺大佬提了个PR，看看会不会合，别人再蛋疼一遍也没啥意义。
