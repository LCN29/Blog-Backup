# [SpringCloud] Gateway 简单使用

spring cloud 的网关功能在低版本是通过 Netflix 的 zuul 实现的，但是到了高版本的 SpringCloud，使用的是 Spring 团队自己开发的 gateway，所以这里就不整理 zuul 的使用了，直接上 gateway。 


## 前提说明
因为 SpirngBoot，SpringCloud 的各个版本之间差异还是挺大的，所以在参照本博客进行学习时，有可能出现因为版本不一致，而出现不同的问题。如果可以和本项目使用的环境保持一致，即使不一致，也尽可能不要跨大版本。
>1. jdk8
>2. SpringBoot : 2.1.4.RELEASE
>3. SpringCloud : Greenwich.SR1
>4. Maven 多模块

## 准备
参考一下 [这个](https://blog.csdn.net/LCN29/article/details/102087449) 里面的环境准备，搭建出2个服务端，1个服务提供者，1个网关服务(也是一个eureka的客户端)

然后在服务提供者，提供一个 REST API
```java
@RestController
public class ServiceController {

    @GetMapping("/server/{num}")
    public String service(@PathVariable("num")int num) {
        String resp = "服务提供端收到了消息:" + num;
        System.out.println("服务提供者进行了服务");
        return resp;
    }
}
```

## 网关服务 依赖
```xml
<dependencies>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-gateway</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    </dependency>
</dependencies>
```
引入 eureka 的客户端，让其可以被 eureka 进行观察，gateway是基于 netty + webflux 实现的，所以本身已经支持 web 功能，可以不用 web 相关的依赖

## 启动类注解
```java
@EnableEurekaClient
@SpringBootApplication
public class GatewayService {

    public static void main(String[] args) {
        SpringApplication.run(GatewayService.class，args);
    }
}
```
我们这里把网关也交由 eureka 管理，所以注解上 客户端相关的注解

## 配置我们的路由规则
```yml
spring:
  cloud:
    gateway:
      # 配置路由规则，访问当前网关的 /server/* 会转发请求到 http://localhost:9091/server/*
      routes:
      # 路由Id，需要唯一
      - id: service-com.can.route
        uri: http://localhost:9091
        predicates:
        - Path=/server/*
```
这是如果你通过浏览器访问 `http://你网关的Ip:端口/server/1` 这个请求经过网关后会被转到 `localhost:9091/server`

## 路由的配置通过代码进行配置
```java
@Configuration
public class RouteConfigruation {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        /**
         * 配置了一个 id 为 second_route，访问 /second/* 会被重定向到 http://localhost:9091/second/*
         */
        return builder.routes()
                .route("second_route"，r -> r.path("/second/*").uri("http://localhost:9091/second/*"))
                .build();
    }
}
```

## 路由网关支持的几种配置

| 关键字 | 作用 | 示例 |
|:- | :-: | :-|
| After | 请求在指定的时间后 | - After=2018-01-20T06:06:06+08:00[Asia/Shanghai] |
|Before | 请求在指定的时间前| - Before=2018-01-20T06:06:06+08:00[Asia/Shanghai]|
|Between | 请求在指定的时间内| - Between=2018-01-20T06:06:06+08:00[Asia/Shanghai]，2019-01-20T06:06:06+08:00[Asia/Shanghai] |
|Cookie | 请求必须带某个key的Cookie，value 是一个正则表达式 | - Cookie=key，regularExpression|
| Header |请求头必须带某个key，value 是一个正则表达式| - Header=X-Request-Id，\d+|
| Host |请求来至于对应的主机，才做处理，通过正则配置| - Host=**.baidu.com|
|Mehtod | 请求的方式限制 | -Method=GET|
| Path | 请求路径匹配，也是一个正则 | - Path=/server/{num}|
| Query | 请求包括指定的请求参数，同时支持指定参数(正则设置)的匹配|- Query=arg 或者 - Query=key，regularExpression|
|RemoteAddr | 请求来自于某个Ip | - RemoteAddr=192.168.1.1/24 |

上面的规则不仅可以单独使用，也可以进行组合使用

## 代码
[GitHub](https://github.com/LCN29/SpringCloud/tree/master/spring-cloud-gateway)
