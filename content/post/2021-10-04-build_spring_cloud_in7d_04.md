+++
title       = "七天构建SpringCloud集群"
subtitle    = "day4：集群通信"
description = "服务调用、文档集中"
author      = "月白"
date        = 2021-10-04T19:00:00+08:00
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

## 一、服务调用

### 1.在bizfacade中开发feignClient

bizfacade添加gradle依赖

```groovy
implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
```

创建feignClient

```java
public interface BizFacadeConst {
    String SVC_UAA = "svc-uaa";
    String APP_BIZ = "app-biz";
    String APP_BIZAGG = "app-bizagg";
}

@FeignClient(name = BizFacadeConst.APP_BIZ, fallbackFactory = BizFeignClientFallbackFactory.class)
public interface BizFeignClient {
    @GetMapping("/demo/appName")
    String appName();
}

@Slf4j
@Component
public class BizFeignClientFallbackFactory implements FallbackFactory<BizFeignClient> {
    @Override
    public BizFeignClient create(Throwable cause) {
        return new BizFeignClient() {
            @Override
            public String appName() {
                log.error("feign error", cause);
                return "biz-fallback";
            }
        };
    }
}
```

### 2.在bizagg中引入bizfacade实现应用调用

bizagg添加gradle依赖

```groovy
implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
implementation project(':app:jiuzhou-bizfacade')
```

application.properties添加配置

```properties
feign.client.config.default.logger-level = full
feign.circuitbreaker.enabled=true
```

启动类增加feign开启注解@EnableFeignClients以及修正包扫描配置，此处需要扫描到bizfacade中的fallbackFactory的Component

```java
@EnableFeignClients(basePackages = "com.yuebaix.jiuzhou.app.bizfacade")
@EnableDiscoveryClient
@SpringBootApplication(scanBasePackages = "com.yuebaix.jiuzhou")
public class BizAggApp {
    public static void main(String[] args) {
        SpringApplication.run(BizAggApp.class, args);
    }
}
```

添加测试接口

```java
@Api(tags = "示例接口")
@Slf4j
@RestController
@RequestMapping("/demo")
public class DemoController {
    @Value("${spring.application.name}")
    private String appName;
    @Autowired
    private UaaFeignClient uaaFeignClient;
    @Autowired
    private BizFeignClient bizFeignClient;

    @ApiOperation("服务名称")
    @GetMapping("/appName")
    public String appName() {
        return appName;
    }

    @ApiOperation("调用svc-uaa")
    @GetMapping("/callSvcUaa")
    public String callSvcUaa() {
        return uaaFeignClient.appName();
    }

    @ApiOperation("调用app-biz")
    @GetMapping("/callAppBiz")
    public String callAppBiz() {
        return bizFeignClient.appName();
    }
}
```

进行调用测试

![](/pic/2021_10_04/call_fallback.png)

![](/pic/2021_10_04/call_fallback_error.png)

![](/pic/2021_10_04/call_success.png)

### 3.在biz与bizclient中引入feign实现

由于调用层级关系，controller有所不痛

```java
//biz中
@Api(tags = "示例接口")
@Slf4j
@RestController
@RequestMapping("/demo")
public class DemoController {
    @Value("${spring.application.name}")
    private String appName;
    @Autowired
    private UaaFeignClient uaaFeignClient;

    @ApiOperation("服务名称")
    @GetMapping("/appName")
    public String appName() {
        return appName;
    }

    @ApiOperation("调用svc-uaa")
    @GetMapping("/callSvcUaa")
    public String callSvcUaa() {
        return uaaFeignClient.appName();
    }
}
```

```java
//bizclient中
@Api(tags = "示例接口")
@Slf4j
@RestController
@RequestMapping("/demo")
public class DemoController {
    @Value("${spring.application.name:app-bizclient}")
    private String appName;
    @Autowired
    private UaaFeignClient uaaFeignClient;
    @Autowired
    private BizFeignClient bizFeignClient;
    @Autowired
    private BizAggFeignClient bizAggFeignClient;

    @ApiOperation("服务名称")
    @GetMapping("/appName")
    public String appName() {
        return appName;
    }

    @ApiOperation("调用svc-uaa")
    @GetMapping("/callSvcUaa")
    public String callSvcUaa() {
        return uaaFeignClient.appName();
    }

    @ApiOperation("调用app-biz")
    @GetMapping("/callAppBiz")
    public String callAppBiz() {
        return bizFeignClient.appName();
    }

    @ApiOperation("调用app-bizagg")
    @GetMapping("/callAppBizagg")
    public String callAppBizagg() {
        return bizAggFeignClient.appName();
    }
}
```

## 二、文档集中

### 1.完成sys-gate对服务的路由

### 2.完成swagger接口的路由配置

## 三、简单登陆



## 四、开发日志


