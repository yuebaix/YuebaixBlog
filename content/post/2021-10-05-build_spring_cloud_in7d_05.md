+++
title       = "七天构建SpringCloud集群"
subtitle    = "day5：安全启动"
description = "基本安全、聚合文档安全访问"
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

最后将变更发布到所有应用上

## 二、网关访问

本来以为spring security简单登陆也是用header来完成的，结果用下来是用账号密码交换cookie去访问的，每个应用的session都是本地的，并没有进行共享，
而且同样从网关访问，用的cookie id又是一样的，无法使用这种方式来进行访问。好在所有Basic Auth都是支持header访问的，很多数据库的账号密码也是通过
这种方式访问的。

Basic Auth的Header生成很简单:

```text
Authorization: Basic base64({username}:{password})
```

在这个例子中：base64要加密的内容为***yuebaix:yuebaix***，所以我们随便在网上找一个在线base64加密可得：eXVlYmFpeDp5dWViYWl4

所以我们要在请求中添加的header内容为：

```text
Authorization: Basic eXVlYmFpeDp5dWViYWl4
```

所以我们在网关路由中配置AddRequestHeaderFilter

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: route-app-bizagg
          uri: lb://app-bizagg
          predicates:
            - Path=/app-bizagg/**
          filters:
            - AddRequestHeader=Authorization, Basic eXVlYmFpeDp5dWViYWl4
        - id: route-app-biz
          uri: lb://app-biz
          predicates:
            - Path=/app-biz/**
          filters:
            - StripPrefix=1
            - AddRequestHeader=Authorization, Basic eXVlYmFpeDp5dWViYWl4
```

## 三、服务调用

现在网关调用应用已经通了，但是服务之间互相调用还是无权限。需要对服务调用feign进行升级。由于各个服务都要调用，所以代码开发在bizfacade中，新增FeignConfig配置类。

```java
@Configuration
public class FeignConfig {
    @Value("${spring.security.user.name}")
    private String username;
    @Value("${spring.security.user.password}")
    private String password;

    @Bean
    public BasicAuthRequestInterceptor basicAuthRequestInterceptor() {
        return new BasicAuthRequestInterceptor(username, password);
    }
}
```

Voilà 这样我们就基本完成了文档的在线安全以及聚合，也可以基本完成应用安全。但这还不够安全，接下来我们来升级oauth2。

## 四、开发日志

* 1.apollo-quick-start PR 合并了

> 今天宋老师PR review回复了。LGTM，最近很高频看到这个词，恶补了下github黑话，相关内容发在另外一篇文章里了，大家感兴趣可以看看。

![](/pic/2021_10_05/apollo_merged.png)

* 2.spring security的文档一如既往的烂

> 这个团队吧，真是一言难尽啊。我相信随着时间的发展，比之前要好一些了。但是整体来看设计上的硬伤，代码上的硬伤，还有文档的缺失，像个车祸现场，我看了
> 下现在的内容，变动非常大，说是跟之前的版本完全不兼容也不为过。一个本质不复杂的项目硬生生给做成了个超级大项目，再加上之前用spring security做权
> 限框架给我带来的痛苦，可以下个结论：任何情况下，但凡有点追求，都不要用spring security，尽管也许它更安全，但是迭代跟调试带来的无意义折腾完全没
> 有意义。这次授权用例还是用spring security开发，随后看有没有完善的一个做法，最近火起来的sa-token也可以尝试下，不行自己写，uaa、sso、monitor
> 等功能，会做一个完善的系统，更新在hongjun(鸿钧)项目中。

* 3.被一个去国外打工回来的抖音博主忧郁到了

> 看他的频道还有国外的天堂瀑布，真的是很美。但是回到家乡以后伤心的光景还是让人难过，这个世界每个人都被卡在一个边缘，跟谁都合不起来，只能自己孤独着。

<div class="clearfix">
<img align="left" style="display: block" width="300px" src="/pic/2021_10_05/ticktok_touching.png" />
</div>
