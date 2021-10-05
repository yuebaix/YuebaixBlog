+++
title       = "七天构建SpringCloud集群"
subtitle    = "day5：安全启动"
description = "基本安全、授权中心"
author      = "月白"
date        = 2021-10-05T19:00:00+08:00
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

## 一、基本安全

初步建立基本安全机制，在svc-uaa里变更gradle配置

```groovy
implementation 'org.springframework.boot:spring-boot-starter-security'
```

在配置文件application.properties中添加基本账户的用户名密码

```properties
spring.security.user.name=yuebaix
spring.security.user.password=yuebaix
```

进行登陆登出测试

[http://localhost:12101/login](http://localhost:12101/login)

![](/pic/2021_10_05/simple_login.png)

[http://localhost:12101/logout](http://localhost:12101/logout)

![](/pic/2021_10_05/simple_logout.png)

最后将变更发布到除sys-gate以外的所有应用上

## 二、Uaa开发

```

```

## 三、开发日志


