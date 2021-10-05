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

到这里在线文档聚合这个功能就已经完善了。其实还可以实现一个GlobalFilter去添加自定义header，并在swagger插件中对自定义header进行识别转换
inferredServer地址，此处主要为了最小化代码量完成功能。

## 三、开发日志

* 1.apollo-quick-start PR 反馈

> 今日宋老师又有反馈了，要把账号密码的环境变量都带上，并且密码要脱敏显示。也许我开发自己的代码太多了，做东西比较随意，我一度也以为这个项目也只是个
> 用来测试的东西。接触下来感觉好像不是，要更认真地面对功能与产品设计，因为这些是一开始就应该能想到的。在apollo这样的项目里，用户非常多，也更应该
> 认真来应对问题。也由于在这个上面改来改去调来调去的，导致自己jiuzhou项目开发进度变慢了。准备明天爆肝一下。

* 2.hugo-theme-cleanwhite 反馈

> 作者把 _谁在使用_ 置顶到issue了，跑过去第一个登记了自己 😉 'Glad to be helpful'

* 3.国庆过得好快

> 也许是因为自己一直在写代码，感觉这几天过得好快，人总会有各种各样的事情缠着你。想要在有限的时间去做一些真正有意义的事情，工作这么多年，实际对社会
> 有贡献，对人生有意义，对大家有价值的东西太少了。人活着不该只为了钱对吗？何况自己依旧孑然一身。问自己一个问题，这辈子做成了什么事情能让自己年老时
> 不至于羞愤懊悔。我想，社会把你绑在车轮上，但你始终有选择的余地不是吗？也许不是世界太嘈杂，只是自己不够宁静致远。

* 4.gateway开发体验好了

> 早年尝试使用gateway开发的时候，自己云里雾里的。不知道是因为自己技术变强了还是gateway完善了。不过始终不够开箱即用啊，过几天研究下soul网关(ShenYu)
> 看看能不能贡献些代码，一起做个开箱即用的产品出来。感慨这几年遇到的大神，xxl许雪里、discovery军哥、apollo宋顺、spring4all翟永超、dromara xiaoyu，
> 有的有比较深入的接触，有的只是在线下见过，有的讨论过issue，有的只在群里聊过几句，甚至有些不愿意提及的渣人都没有列出来，所有人做成一个成功的项目，
> 其社区宣传、影响力、作品产品完善程度与生产验证搜集bug驱动产品进步，无一不是把社区产品迭代做得比较好。一个好的产品要具有前瞻性、要足够完善、要bug
> 少可验证，更重要的是，要有弥补空白的定位。如果定位与其他产品重叠，一定要有足够痛的点让人想完成迁移。apache社区号称社区大于产品，有人才能有产品，
> 一个人做多久都不可能完成一个完善的产品，即便可以，那付出的精力不可能是在满足生活前提下完成的。要足够visionary，要足够responsible，更关心产品，
> 更想要表达。

* 5.springfox-oas 居然不支持 X-Forwarded-Prefix

> 这个功能在早前swagger2.0的时候就已经支持了，在swagger3.0的webflux版本看着也是支持的，为什么在这个mvc版本不支持，有点令人疑惑，是不是也应该
> 去提个issue来升级一下这个feature。

> 去看了一下，现在springfox貌似是个印度人在维护，但是PR已经很久没有人合并了，不知道还能不能也不知道啥时候才会再release了，这个项目好像是
> josh long参与发起的，不知道为啥转手出去了。如果这老哥被新冠缠住了，那也太让人难过了。
