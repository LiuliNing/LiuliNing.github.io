# Sentinel 熔断限流学习




<!--more-->
# Sentinel 熔断限流学习

```
Sentinel版本 
	控制台 1.8.2 
	客户端 2.2.7.RELEASE
```

```
参考信息
控制台 https://github.com/alibaba/Sentinel/wiki/%E6%8E%A7%E5%88%B6%E5%8F%B0
官方文档 https://github.com/alibaba/spring-cloud-alibaba/wiki/Sentinel
博客 https://www.cnblogs.com/crazymakercircle/p/14285001.html
```

## 1 控制台搭建

```
# 官网下载控制台Jar文件
https://github.com/alibaba/Sentinel/releases
```

```
# 运行并启动控制台jar文件
java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=sentinel -jar sentinel-dashboard-1.8.2.jar
```

```
windows bat一键启动脚本（替换jar包路径）
@echo off
start cmd /k "cd /d D:\work\springclude\sentinel && java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=sentinel -jar sentinel-dashboard-1.8.2.jar"
exit
```

```
#参数解释
用于指定 Sentinel 控制台端口为 8080
-Dserver.port=8080 
用于控制台对外暴露的服务地址
-Dcsp.sentinel.dashboard.server 
用于指定项目名称
-Dproject.name=sentinel 
用于指定控制台的登录用户名为 sentinel
-Dsentinel.dashboard.auth.username=sentinel 
用于指定控制台的登录密码为 123456；如果省略这两个参数，默认用户和密码均为 sentinel
-Dsentinel.dashboard.auth.password=123456 
用于指定 Spring Boot 服务端 session 的过期时间，如 7200 表示 7200 秒；60m 表示 60 分钟，默认为 30 分钟
-Dserver.servlet.session.timeout=7200 
```

```
# 访问Sentinel 控制台界面 
localhost:8080
```



## 2 环境搭建

### 2.1 引入依赖

```
		<!-- 熔断限流 https://mvnrepository.com/artifact/com.alibaba.cloud/spring-cloud-starter-alibaba-sentinel -->
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
            <version>2.2.7.RELEASE</version>
        </dependency>
```

### 2.2 配置文件

```
spring:
  # 熔断限流
  cloud:
    sentinel:
      transport:
      	# sentinel控制台地址
        dashboard: localhost:8080
        # 应用与Sentinel控制台交互的端口，应用本地会起一个该端口占用的HttpServer
        port: 8081
      # 是否提前触发 Sentinel 初始化
      eager: true
```

### 2.3 持久化配置至nacos（TODO）

```
Sentinel Dashboard中添加的规则是存储在内存中的，只要项目一重启规则就丢失了，需要规则持久化到nacos中
参考博客
https://blog.csdn.net/h273979586/article/details/115596602?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.nonecase
```

[持久化配置至nacos]: https://blog.csdn.net/h273979586/article/details/115596602?spm=1001.2101.3001.6650.1&amp;utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.nonecase&amp;depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7Edefault-1.nonecase



## 3 常用规则配置说明

Spring Cloud Alibaba Sentinel 提供了这些配置选项:

| 配置项                                                  | 含义                                                         | 默认值            |
| ------------------------------------------------------- | ------------------------------------------------------------ | ----------------- |
| `spring.application.name` or `project.name`             | Sentinel项目名                                               |                   |
| `spring.cloud.sentinel.enabled`                         | Sentinel自动化配置是否生效                                   | true              |
| `spring.cloud.sentinel.eager`                           | 是否提前触发 Sentinel 初始化                                 | false             |
| `spring.cloud.sentinel.transport.port`                  | 应用与Sentinel控制台交互的端口，应用本地会起一个该端口占用的HttpServer | 8719              |
| `spring.cloud.sentinel.transport.dashboard`             | Sentinel 控制台地址                                          |                   |
| `spring.cloud.sentinel.transport.heartbeat-interval-ms` | 应用与Sentinel控制台的心跳间隔时间                           |                   |
| `spring.cloud.sentinel.transport.client-ip`             | 此配置的客户端IP将被注册到 Sentinel Server 端                |                   |
| `spring.cloud.sentinel.filter.order`                    | Servlet Filter的加载顺序。Starter内部会构造这个filter        | Integer.MIN_VALUE |
| `spring.cloud.sentinel.filter.url-patterns`             | 数据类型是数组。表示Servlet Filter的url pattern集合          | /*                |
| `spring.cloud.sentinel.filter.enabled`                  | Enable to instance CommonFilter                              | true              |
| `spring.cloud.sentinel.metric.charset`                  | metric文件字符集                                             | UTF-8             |
| `spring.cloud.sentinel.metric.file-single-size`         | Sentinel metric 单个文件的大小                               |                   |
| `spring.cloud.sentinel.metric.file-total-count`         | Sentinel metric 总文件数量                                   |                   |
| `spring.cloud.sentinel.log.dir`                         | Sentinel 日志文件所在的目录                                  |                   |
| `spring.cloud.sentinel.log.switch-pid`                  | Sentinel 日志文件名是否需要带上 pid                          | false             |
| `spring.cloud.sentinel.servlet.block-page`              | 自定义的跳转 URL，当请求被限流时会自动跳转至设定好的 URL     |                   |
| `spring.cloud.sentinel.flow.cold-factor`                | WarmUp 模式中的 冷启动因子                                   | 3                 |
| `spring.cloud.sentinel.zuul.order.pre`                  | SentinelZuulPreFilter 的 order                               | 10000             |
| `spring.cloud.sentinel.zuul.order.post`                 | SentinelZuulPostFilter 的 order                              | 1000              |
| `spring.cloud.sentinel.zuul.order.error`                | SentinelZuulErrorFilter 的 order                             | -1                |
| `spring.cloud.sentinel.scg.fallback.mode`               | Spring Cloud Gateway 流控处理逻辑 (选择 `redirect` or `response`) |                   |
| `spring.cloud.sentinel.scg.fallback.redirect`           | Spring Cloud Gateway 响应模式为 'redirect' 模式对应的重定向 URL |                   |
| `spring.cloud.sentinel.scg.fallback.response-body`      | Spring Cloud Gateway 响应模式为 'response' 模式对应的响应内容 |                   |
| `spring.cloud.sentinel.scg.fallback.response-status`    | Spring Cloud Gateway 响应模式为 'response' 模式对应的响应码  | 429               |
| `spring.cloud.sentinel.scg.fallback.content-type`       | Spring Cloud Gateway 响应模式为 'response' 模式对应的 content-type | application/json  |

```
请注意。这些配置只有在 Servlet 环境下才会生效，RestTemplate 和 Feign 针对这些配置都无法生效
```



## 4 核心概念

```
使用 Sentinel 来进行熔断保护，主要分为两个步骤：定义资源和定义规则。
```

### 4.1.定义资源

```
	资源：可以是任何东西，一个服务，服务里的方法，甚至是一段代码。
	资源是 Sentinel 的关键概念。它可以是 Java 应用程序中的任何内容，例如，由应用程序提供的服务，或由应用程序调用的其它应用提供的服务，RPC接口方法，甚至可以是一段代码。
	只要通过 Sentinel API 定义的代码，就是资源，能够被 Sentinel 保护起来。大部分情况下，可以使用方法签名，URL，甚至服务名称作为资源名来标示资源。
	把需要控制流量的代码用 Sentinel的关键代码 SphU.entry("资源名") 和 entry.exit() 包围起来即可。

示例代码1
    Entry entry = null;
    try {
        // 定义一个sentinel保护的资源，名称为test-sentinel-api
        entry = SphU.entry(resourceName);
        // 模拟执行被保护的业务逻辑耗时
        Thread.sleep(100);
        return a;
    } catch (BlockException e) {
        // 如果被保护的资源被限流或者降级了，就会抛出BlockException
        log.warn("资源被限流或降级了", e);
        return "资源被限流或降级了";
    } catch (InterruptedException e) {
        return "发生InterruptedException";
    } finally {
        if (entry != null) {
            entry.exit();
        }
 
        ContextUtil.exit();
    }
}

示例代码2
public static void main(String[] args) {
    // 配置规则.
    initFlowRules();

    while (true) {
        // 1.5.0 版本开始可以直接利用 try-with-resources 特性
        try (Entry entry = SphU.entry("HelloWorld")) {
            // 被保护的逻辑
            System.out.println("hello world");
	} catch (BlockException ex) {
            // 处理被流控的逻辑
	    System.out.println("blocked!");
	}
    }
}
示例代码3 （推荐使用）（注意：注解方式埋点不支持 private 方法。）
@SentinelResource("HelloWorld")

public void helloWorld() {
    // 资源中的逻辑
    System.out.println("hello world");
}

```



```
@SentinelResource 用于定义资源，并提供可选的异常处理和 fallback 配置项
使用说明见官方文档
https://github.com/alibaba/Sentinel/wiki/%E6%B3%A8%E8%A7%A3%E6%94%AF%E6%8C%81
（以下说明为复制）
value：资源名称，必需项（不能为空）

entryType：entry 类型，可选项（默认为 EntryType.OUT）

blockHandler / blockHandlerClass: blockHandler 对应处理 BlockException 的函数名称，可选项。blockHandler 函数访问范围需要是 public，返回类型需要与原方法相匹配，参数类型需要和原方法相匹配并且最后加一个额外的参数，类型为 BlockException。blockHandler 函数默认需要和原方法在同一个类中。若希望使用其他类的函数，则可以指定 blockHandlerClass 为对应的类的 Class 对象，注意对应的函数必需为 static 函数，否则无法解析。

fallback / fallbackClass：fallback 函数名称，可选项，用于在抛出异常的时候提供 fallback 处理逻辑。fallback 函数可以针对所有类型的异常（除了 exceptionsToIgnore 里面排除掉的异常类型）进行处理。fallback 函数签名和位置要求：
返回值类型必须与原函数返回值类型一致；
方法参数列表需要和原函数一致，或者可以额外多一个 Throwable 类型的参数用于接收对应的异常。
fallback 函数默认需要和原方法在同一个类中。若希望使用其他类的函数，则可以指定 fallbackClass 为对应的类的 Class 对象，注意对应的函数必需为 static 函数，否则无法解析。

defaultFallback（since 1.6.0）：默认的 fallback 函数名称，可选项，通常用于通用的 fallback 逻辑（即可以用于很多服务或方法）。默认 fallback 函数可以针对所有类型的异常（除了 exceptionsToIgnore 里面排除掉的异常类型）进行处理。若同时配置了 fallback 和 defaultFallback，则只有 fallback 会生效。defaultFallback 函数签名要求：
返回值类型必须与原函数返回值类型一致；
方法参数列表需要为空，或者可以额外多一个 Throwable 类型的参数用于接收对应的异常。
defaultFallback 函数默认需要和原方法在同一个类中。若希望使用其他类的函数，则可以指定 fallbackClass 为对应的类的 Class 对象，注意对应的函数必需为 static 函数，否则无法解析。

exceptionsToIgnore（since 1.6.0）：用于指定哪些异常被排除掉，不会计入异常统计中，也不会进入 fallback 逻辑中，而是会原样抛出。
1.8.0 版本开始，defaultFallback 支持在类级别进行配置。

注：1.6.0 之前的版本 fallback 函数只针对降级异常（DegradeException）进行处理，不能针对业务异常进行处理。

特别地，若 blockHandler 和 fallback 都进行了配置，则被限流降级而抛出 BlockException 时只会进入 blockHandler 处理逻辑。若未配置 blockHandler、fallback 和 defaultFallback，则被限流降级时会将 BlockException 直接抛出（若方法本身未定义 throws BlockException 则会被 JVM 包装一层 UndeclaredThrowableException）。

```



### 4.2.定义规则

```
Sentinel 支持以下几种规则：流量控制规则、熔断降级规则、系统保护规则、来源访问控制规则、热点参数规则
```

#### 4.2.1 熔断降级

```
熔断降级对调用链路中不稳定的资源进行熔断降级是保障高可用的重要措施之一。

由于调用关系的复杂性，如果调用链路中的某个资源不稳定，最终会导致请求发生堆积。Sentinel 熔断降级会在调用链路中某个资源出现不稳定状态时（例如调用超时或异常比例升高），对这个资源的调用进行限制，让请求快速失败，避免影响到其它的资源而导致级联错误。当资源被降级后，在接下来的降级时间窗口之内，对该资源的调用都自动熔断（默认行为是抛出 DegradeException）
```

| Field              | 说明                                                         | 默认值     |
| ------------------ | ------------------------------------------------------------ | ---------- |
| resource           | 资源名，即规则的作用对象                                     |            |
| grade              | 熔断策略，支持慢调用比例/异常比例/异常数策略                 | 慢调用比例 |
| count              | 慢调用比例模式下为慢调用临界 RT（超出该值计为慢调用）；异常比例/异常数模式下为对应的阈值 |            |
| timeWindow         | 熔断时长，单位为 s                                           |            |
| minRequestAmount   | 熔断触发的最小请求数，请求数小于该值时即使异常比率超出阈值也不会熔断（1.7.0 引入） | 5          |
| statIntervalMs     | 统计时长（单位为 ms），如 60*1000 代表分钟级（1.8.0 引入）   | 1000 ms    |
| slowRatioThreshold | 慢调用比例阈值，仅慢调用比例模式有效（1.8.0 引入）           |            |

通常用以下几种降级策略：

- 平均响应时间 (DEGRADE_GRADE_RT)：

  当资源的平均响应时间超过阈值（DegradeRule 中的 count，以 ms 为单位）之后，资源进入准降级状态。如果接下来 1s 内持续进入 5 个请求（即 QPS >= 5），它们的 RT 都持续超过这个阈值，那么在接下的时间窗口（DegradeRule 中的 timeWindow，以 s 为单位）之内，对这个方法的调用都会自动地熔断（抛出 DegradeException）。

  > 注意 Sentinel 默认统计的 RT 上限是 4900 ms，超出此阈值的都会算作 4900 ms，若需要变更此上限可以通过启动配置项 -Dcsp.sentinel.statistic.max.rt=xxx 来配置。、

  

- 异常比例 (DEGRADE_GRADE_EXCEPTION_RATIO)：

  当资源的每秒异常总数占通过量的比值超过阈值（DegradeRule 中的 count）之后，资源进入降级状态，即在接下的时间窗口（DegradeRule 中的 timeWindow，以 s 为单位）之内，对这个方法的调用都会自动地返回。

  > 异常比率的阈值范围是 [0.0, 1.0]，代表 0% - 100%。

  

- 异常数 (DEGRADE_GRADE_EXCEPTION_COUNT)：

  当资源近 1 分钟的异常数目超过阈值之后会进行熔断。

  > 注意由于统计时间窗口是分钟级别的，若 timeWindow 小于 60s，则结束熔断状态后仍可能再进入熔断状态。

#### 4.2.2 限流

```
流量控制(Flow Control)，原理是监控应用流量的QPS或并发线程数等指标，当达到指定阈值时对流量进行控制，避免系统被瞬时的流量高峰冲垮，保障应用高可用性。
```

一条限流规则主要由下面几个因素组成，我们可以组合这些元素来实现不同的限流效果：

| Field           | 说明                                                         | 默认值                        |
| :-------------- | :----------------------------------------------------------- | :---------------------------- |
| resource        | 资源名，资源名是限流规则的作用对象                           |                               |
| count           | 限流阈值                                                     |                               |
| grade           | 限流阈值类型，QPS 或线程数模式                               | QPS 模式                      |
| limitApp        | 流控针对的调用来源                                           | `default`，代表不区分调用来源 |
| strategy        | 判断的根据是资源自身，还是根据其它关联资源 (`refResource`)，还是根据链路入口 | 根据资源本身                  |
| controlBehavior | 流控效果（直接拒绝 / 排队等待 / 慢启动模式）                 | 直接拒绝                      |

```
资源名：唯一名称，默认请求路径
针对来源：Sentinel可以针对调用者进行限流，填写微服务名，默认为default(不区分来源)
阈值类型/单机阈值：
1.QPS：每秒请求数，当前调用该api的QPS到达阈值的时候进行限流
2.线程数：当调用该api的线程数到达阈值的时候，进行限流
是否集群：是否为集群
```

```
流控的三种模式
1.直接：当api大达到限流条件时，直接限流
2.关联：当关联的资源到达阈值，就限流自己
3.链路：只记录指定路上的流量，指定资源从入口资源进来的流量，如果达到阈值，就进行限流，api级别的限流
```

#### 4.2.3 热点限流

| Field             | 说明                                                         | 默认值   |
| ----------------- | ------------------------------------------------------------ | -------- |
| resource          | 资源名，必填                                                 |          |
| count             | 限流阈值，必填                                               |          |
| grade             | 限流模式                                                     | QPS 模式 |
| durationInSec     | 统计窗口时间长度（单位为秒），1.6.0 版本开始支持             | 1s       |
| controlBehavior   | 流控效果（支持快速失败和匀速排队模式），1.6.0 版本开始支持   | 快速失败 |
| maxQueueingTimeMs | 最大排队等待时长（仅在匀速排队模式生效），1.6.0 版本开始支持 | 0ms      |
| paramIdx          | 热点参数的索引，必填，对应 SphU.entry(xxx, args) 中的参数索引位置 |          |
| paramFlowItemList | 参数例外项，可以针对指定的参数值单独设置限流阈值，不受前面 count 阈值的限制。仅支持基本类型和字符串类型 |          |
| clusterMode       | 是否是集群参数流控规则                                       | false    |
| clusterConfig     | 集群流控相关配置                                             |          |
|                   |                                                              |          |

#### 4.2.4 系统保护

```
系统规则支持以下的模式：
Load 自适应（仅对 Linux/Unix-like 机器生效）：系统的 load1 作为启发指标，进行自适应系统保护。当系统 load1 超过设定的启发值，且系统当前的并发线程数超过估算的系统容量时才会触发系统保护（BBR 阶段）。系统容量由系统的 

maxQps * minRt 估算得出。设定参考值一般是 CPU cores * 2.5。

CPU usage（1.5.0+ 版本）：当系统 CPU 使用率超过阈值即触发系统保护（取值范围 0.0-1.0），比较灵敏。

平均 RT：当单台机器上所有入口流量的平均 RT 达到阈值即触发系统保护，单位是毫秒。

并发线程数：当单台机器上所有入口流量的并发线程数达到阈值即触发系统保护。

入口 QPS：当单台机器上所有入口流量的 QPS 达到阈值即触发系统保护。
```

#### 4.2.5 黑白名单

```
授权规则，即黑白名单规则（AuthorityRule）非常简单，主要有以下配置项：

resource：资源名，即限流规则的作用对象
limitApp：对应的黑名单/白名单，不同 origin 用 , 分隔，如 appA,appB
strategy：限制模式，AUTHORITY_WHITE 为白名单模式，AUTHORITY_BLACK 为黑名单模式，默认为白名单模式
```



##  采坑记录

```
1.控制台配置规则后，无法生效，也不显示配置记录
本地项目fastjson版本过低，为1.1.7,升级为1.2.75,问题解决
    <properties>
        <fastjson.version>1.2.75</fastjson.version>
    </properties>
        <!-- fastjson https://mvnrepository.com/artifact/com.alibaba/fastjson -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>${fastjson.version}</version>
        </dependency>

```


