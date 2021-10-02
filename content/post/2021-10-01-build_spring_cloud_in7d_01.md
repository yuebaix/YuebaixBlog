+++
title       = "七天构建SpringCloud集群"
subtitle    = "day1：系统设计"
description = "背景目标、知识准备、系统设计、实施步骤"
author      = "月白"
date        = 2021-10-01T19:00:00+08:00
URL         = ""
image       = ""
categories  = ["TECH"]
tags        = ["干货", "教程"]
showtoc     = true
+++

项目地址：[jiuzhou (九州) https://github.com/yuebaix/jiuzhou](https://github.com/yuebaix/jiuzhou)

<a style="display: inline-block;width: 400px;height: 170px" target="_blank" href="https://github.com/yuebaix/jiuzhou">
    <img align="left" src="https://github-readme-stats.vercel.app/api/pin/?username=yuebaix&theme=highcontrast&repo=jiuzhou" />
</a>

## 一、背景目标

之前构建的 spring cloud 集群版本已经比较老了，现在spring cloud已经更新了 G、H、2020 系列版本。现重新构建一个典型集群用于示意spring cloud的典型架构，并尝试一些新特性。

## 二、知识准备

由于个人搭建集群，用单机模拟集群，受搭建环境限制与便于分发，要使用到docker-compose来构建本地环境，需要准备的知识如下：

* docker
* spring cloud
* zookeeper
* ctrip apollo
* xxljob
* apisix
* uaa oauth2

## 三、系统设计

系统服务分层：sys、svc、app

单服务应用代码分层：dao、service、scene、controller

单服务应用功能分层：job、admin、svc、client、api

## 四、实施步骤

* 1.安装环境
* 2.搭建中间件
* 3.对接服务注册、配置中心
* 4.构建网关、在线文档集中
* 5.构建授权中心、各节点接入
