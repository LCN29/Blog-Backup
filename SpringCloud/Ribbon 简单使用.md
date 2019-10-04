# [SpringCloud] Ribbon 简单使用

## 前提说明
因为 SpirngBoot，SpringCloud 的各个版本之间差异还是挺大的，所以在参照本博客进行学习时，有可能出现因为版本不一致，而出现不同的问题。如果可以和本项目使用的环境保持一致，即使不一致，也尽可能不要跨大版本。
>1. jdk8
>2. SpringBoot : 2.1.4.RELEASE
>3. SpringCloud : Greenwich.SR1
>4. Maven 多模块

## 准备
1. 先建立一个 maven 的父模块，也就是整个项目里面只有一个 pom 文件
2. 在父 pom 里面添加一些 共用的配置，比如 SpringBoot，SpringCloud 的依赖，可以参考这一个:  [Pom](https://github.com/LCN29/SpringCloud/blob/master/spring-cloud-eureka/pom.xml "父pom配置")
3. 同时我们需要了能体现效果，我们需要准备2个服务端，2个服务提供方，11个服务调用方

## 搭建2个服务端

1 服务端的子模块的依赖，启动类，配置可参照这个: [Eureka 简单使用](https://blog.csdn.net/LCN29/article/details/102019053) 中 `建立一个单机的 Eureka 服务端`  

2 在resources里面 新建2个新的配置文件 `application-8081.yml` 和 `application-8082.yml`，那么现在有3个配置文件了，配置的内容可参考一下 [这里](https://github.com/LCN29/SpringCloud/tree/master/spring-cloud-ribbon/server-eureka/src/main/resources)

3 为了让我们的子模块能够启动为2个服务端，需要我们做一下启动的配置
>1. Idea 找到 右上角的 这个  
![Alt '配置'](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9zMi5heDF4LmNvbS8yMDE5LzEwLzA0L3VETXpLZi5wbmc?x-oss-process=image/format,png)  

>2. 点一下左边的 绿色 +，找到 SpringBoot 项，配置一下 下面的三项
![Alt '配置'](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9zMi5heDF4LmNvbS8yMDE5LzEwLzA0L3VEUVBhUS5wbmc?x-oss-process=image/format,png)

>3. Program arguments 配置的内容为 `--spring.profiles.active=8081`，作用是启动 这个 Java 程序使用的环境是 `8081`环境，也就是对应了我们的 `application-8081。yml`里面的配置

>4. 同理，我们在配置多一条 `8082`环境的启动命令，这样我们的子模块就能启动2这样我们的子模块就能启动2个程序了

## 搭建2个服务提供方
1 服务提供方的子模块的依赖，启动类，配置可参照这个:  [Eureka 简单使用](https://blog.csdn.net/LCN29/article/details/102019053) 中 `建立一个 Eureka 客户端`

2 同样在resources里面新建2个新的配置文件 `application-9091.yml` 和 `application-9092.yml`，3个配置文件的内容，可以参考一下 [这里](https://github.com/LCN29/SpringCloud/tree/master/spring-cloud-ribbon/client-provider/src/main/resources)

3 配置我们的服务提供方的启动命令行能启动2个命令，和服务端的类似

4 编写我们的服务提供方提供的服务
```java
@RestController
public class ProviderController {

    @GetMapping("/hello")
    public String hello(HttpServletRequest request) {
        //打印请求的地址
        String resp = " 当前响应来自于" + request.getRequestURL();
        return resp;
    }
}
```

## 搭建1个服务的调用方
参照 [Eureka 简单使用](https://blog.csdn.net/LCN29/article/details/102019053) 中 `Eureka 客户端间服务调用` 进行 `Feign`的配置

至此，我们的环境就搭好了，项目如果正常的话，这时候，服务调用方是可以调用到服务提供方的，但是没法负载

## Ribbon 负载均衡
1 在服务调用方加上依赖
```xml
<!-- 负载均衡 -->
<dependency>
   <groupId>org.springframework.cloud</groupId>
   <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
```
这时候，启动你的项目，通过浏览器访问你的服务调用者，你会发现你的服务调用者已经在轮流调用2个服务提供者了。 说明 Ribbon 已经起作用了，Ribbon 默认的负载算法就是 轮询

2 如果你想要修改 Ribbon 的负载算法，可以通过配置文件
```yml
# 这里为服务提供者的应用名
client-provider:
  ribbon:
    # 随机算法，
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule
```
| 策略对应的实现类| 策略描述|
| :- | :-: |
| com.netflix.loadbalancer.RoundRobinRule| 轮询选择server|
| com.netflix.loadbalancer.RandomRule | 随机选择一个server|
| com.netflix.loadbalancer.BestAvailableRule| 选择一个最小的并发请求的server|
| com.netflix.loadbalancer.AvailabilityFilteringRule| 过滤掉那些因为一直连接失败的被标记为circuit tripped的后端server，并过滤掉那些高并发的的后端server（active connections 超过配置的阈值）|
|com.netflix.loadbalancer.WeightedResponseTimeRule| 根据响应时间分配一个weight，响应时间越长，weight越小，被选中的可能性越低|
|com.netflix.loadbalancer.RetryRule|先按照RoundRobinRule的策略获取服务，如果获取失败则在制定时间内进行重试，获取可用的服务|
| com.netflix.loadbalancer.ZoneAvoidanceRule|复合判断server所在区域的性能和server的可用性选择server|

3 ribbon 同时还支持代码形式的配置
>1. 新建一个配置类
```java
public class RibbonRuleConfig {
    @Bean
    public IRule ribbonRule() {
        // 返回我们需要的负载策略
        return new RandomRule();
    }
}
```
>2. 指定调用方在调用哪个应用时，使用什么策略
```java
// 调用 client-provider 的服务时，使用  RibbonRuleConfig 配置的策略
@RibbonClient(name = "client-provider"，configuration = RibbonRuleConfig.class)
public class MyRibbonClient {
}
```

4 ribbon 同时支持自定义负载策略，实现 AbstractLoadBalancerRule 接口，就可以了
```java
public class RibbonRule extends AbstractLoadBalancerRule {

    @Override
    public void initWithNiwsConfig(IClientConfig iClientConfig) {
    }

    @Override
    public Server choose(Object o) {
        // 实现你的逻辑，最后返回选择的服务实例
        return null;
    }
}
```

## 代码
[GitHub](https://github.com/LCN29/SpringCloud/tree/master/spring-cloud-ribbon)
