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

由于调用层级关系，controller有所不同

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

打开gate配置文件application.properties调试配置方便查看转发情况

```properties
logging.level.org.springframework.cloud.gateway=trace
spring.cloud.gateway.httpclient.wiretap=true
spring.cloud.gateway.httpserver.wiretap=true
```

properties配置路由比较suffer，干脆再加个application.yml来配置路由地址

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
            - StripPrefix=1
        - id: route-app-biz
          uri: lb://app-biz
          predicates:
            - Path=/app-biz/**
          filters:
            - StripPrefix=1
```

再提供一下application.properties路由的配置示例

```properties
spring.cloud.gateway.routes[0].id=app-bizagg-route
spring.cloud.gateway.routes[0].uri=lb://app-bizagg
spring.cloud.gateway.routes[0].predicates[0]=Path=/app-bizagg/**
spring.cloud.gateway.routes[0].filters[0]=StripPrefix=1
```

这个时候gate对各个服务的请求就打通了，但是返回的文档内容没有前缀，无法访问到服务上，如果路由前缀跟服务contextPath一致的话应该已经通了，但是这里
没有设置contextPath，比较受限，一会来解决这个问题。

### 2.完成swagger接口的路由配置

先来解决文档/v3/api-docs返回的json中server地址无前缀的问题

> 在线文档接口前缀可以自动根据请求地址来自动适配，原理是网关会将请求头*X-Forwarded-Prefix*进行叠加转发，springfox文档会自动根据这个header来
> 进行适配。此次开发中，发现有两个变化，一个是gateway会自动将这个header在stripPrefix的插件时自动转发下来；一个是升级open api以后，springfox
> 自己的mvc实现的controller把这个功能移除掉了，需要扩展变更上去才可使用。

先设置*X-Forwarded-Prefix*的转发，实际这个地方默认就是这样的配置。

```properties
spring.cloud.gateway.x-forwarded.prefix-enabled=true
spring.cloud.gateway.x-forwarded.prefix-append=true
```

由于springfox在inferredServer这个地方的处理是用spring plugin实现的，所以我们可以自己实现一个一样的插件在它处理出来的server url上添加前缀。
springfox本身实现的plugin是优先级最高的，所以我们只要实现不配置顺序就可以了。由于各个业务可能都要用，所以就实现在了bizfacade包中。

在bizfacade的gradle文件中新增依赖

```groovy
implementation 'javax.servlet:javax.servlet-api'
implementation 'io.springfox:springfox-boot-starter'
```

在bizfacade中实现插件并导入SpringContext

```java
@Component
public class WebMvcXForwardedPrefixOpenApiTransformationFilter implements WebMvcOpenApiTransformationFilter {
    private static final String X_FORWARDED_PREFIX = "X-Forwarded-Prefix";
    private static final String COMMA = ",";

    @Override
    public OpenAPI transform(OpenApiTransformationContext<HttpServletRequest> context) {
        OpenAPI openApi = context.getSpecification();
        context.request().ifPresent(httpServletRequest -> {
            String xForwardedPrefix = httpServletRequest.getHeader(X_FORWARDED_PREFIX);
            if (!StringUtils.isEmpty(xForwardedPrefix)) {
                String[] prefixArr = xForwardedPrefix.split(COMMA);
                if (prefixArr.length > 0) {
                    String prefix = fixup(prefixArr[0]);
                    List<Server> servers = openApi.getServers();
                    for (Server server : servers) {
                        UriComponentsBuilder uriComponentsBuilder = UriComponentsBuilder.fromUriString(server.getUrl());
                        uriComponentsBuilder.path(prefix);
                        UriComponents uriComponents = uriComponentsBuilder.build();
                        server.setUrl(uriComponents.toUriString());
                    }
                }
            }
        });
        return openApi;
    }

    @Override
    public boolean supports(DocumentationType delimiter) {
        return delimiter == DocumentationType.OAS_30;
    }

    private String fixup(String path) {
        if (StringUtils.isEmpty(path)
                || "/".equals(path)
                || "//".equals(path)) {
            return "/";
        }
        return StringUtils.trimTrailingCharacter(path.replace("//", "/"), '/');
    }
}
```

实现SwaggerResourcesProvider实现路由转换为多definition。一般实现SwaggerResourcesProvider接口即可，这里我是继承InMemorySwaggerResourcesProvider
来实现的，可以把gateway本身的接口也添加近definition中。

```java
@Primary
@Configuration
public class GatewayRoutesOasSwaggerResourcesProvider extends InMemorySwaggerResourcesProvider {
    protected static final String API_DOCS_URI = "/v3/api-docs";
    private static final String SWAGGER_VERSION = "3.0";
    private final RouteLocator routeLocator;
    private final GatewayProperties gatewayProperties;

    public GatewayRoutesOasSwaggerResourcesProvider(
            RouteLocator routeLocator, GatewayProperties gatewayProperties,
            Environment environment, DocumentationCache documentationCache, DocumentationPluginsManager pluginsManager) {
        super(environment, documentationCache, pluginsManager);
        this.routeLocator = routeLocator;
        this.gatewayProperties = gatewayProperties;
    }

    @Override
    public List<SwaggerResource> get() {
        List<SwaggerResource> resources = new ArrayList<>();
        resources.addAll(super.get());
        List<String> routes = new ArrayList<>();
        routeLocator.getRoutes().subscribe(route -> routes.add(route.getId()));
        gatewayProperties.getRoutes().stream().filter(routeDefinition -> routes.contains(routeDefinition.getId())).forEach(route -> {
            route.getPredicates().stream()
                    .filter(predicateDefinition -> ("Path").equalsIgnoreCase(predicateDefinition.getName()))
                    .forEach(predicateDefinition -> resources.add(swaggerResource(route.getId(),
                            predicateDefinition.getArgs().get(NameUtils.GENERATED_NAME_PREFIX + "0")
                                    .replace("**", API_DOCS_URI))));
        });
        return resources;
    }

    private SwaggerResource swaggerResource(String name, String location) {
        SwaggerResource swaggerResource = new SwaggerResource();
        swaggerResource.setName(name);
        swaggerResource.setLocation(location);
        swaggerResource.setSwaggerVersion(SWAGGER_VERSION);
        return swaggerResource;
    }
}
```

### 3.进行综合测试

为方便测试，对biz-agg配置application.properties进行contextPath的调整，可以与biz-app对照

```groovy
server.servlet.context-path=/app-bizagg
```

修改网关对应的转发配置

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: route-app-bizagg
          uri: lb://app-bizagg
          predicates:
            - Path=/app-bizagg/**
#          filters:
#            - StripPrefix=1
```

![](/pic/2021_10_04/gate_knife_doc.png)

![](/pic/2021_10_04/gate_knife_doc_definitions.png)

### 4.总结

到这里在线文档聚合这个功能就已经完善了。其实还可以实现一个GlobalFilter去添加自定义header，并在swagger插件中对自定义header进行识别转换inferredServer地址，此处主要为了最小化代码量完成功能。

## 三、开发日志


