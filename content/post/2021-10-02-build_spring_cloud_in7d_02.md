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

### 2.C盘又爆了

### 3.内存爆了

### 4.被换行符坑了

