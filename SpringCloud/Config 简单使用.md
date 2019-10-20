# [SpringCloud] Config 简单使用

## 前提说明
因为 SpirngBoot，SpringCloud 的各个版本之间差异还是挺大的，所以在参照本博客进行学习时，有可能出现因为版本不一致，而出现不同的问题。如果可以和本项目使用的环境保持一致，即使不一致，也尽可能不要跨大版本。
>1. jdk8
>2. SpringBoot : 2.1.4.RELEASE
>3. SpringCloud : Greenwich.SR1
>4. Maven 多模块

## 准备
>1. 参考一下 [这个](https://blog.csdn.net/LCN29/article/details/102087449) 里面的环境准备，搭建出2个服务端，1个配置服务端，1个配置客户端
>2. 在 Github 上新建一个存放配置用的仓库，我这里的仓库名叫做 `spring-cloud-config`，>3. 在仓库里面新建了一个文件夹 `application-config`，里面新建了三个文件 `config-dev.yml`，`config-test.yml`，`config-prod.yml`
>4. 其中里面的内容类似的，比如 `config-dev.yml`的内容为
```yml
my:
  name: dev
```

## 手动版本
1.服务端引入依赖
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

2.服务端配置文件  
```yml
spring:
  application:
    name: spring-cloud-config-server
  cloud:
    config:
      server:
        git:
          uri: 你GitHub配置仓库的地址
          # 指定某个目录
          search-paths: 配置仓库里面哪个文件夹存放着你这个项目需要的配置文件(这里我的就是 /application-config)
          username: GitHub的用户名
          password: GitHub的登录密码

eureka:
  instance:
    instance-id: ${spring.application.name}:${server.port}
  client:
    service-url:
      # 向服务端的注册地址
      defaultZone: http://eureka-8081.com:8081/eureka/，http://eureka-8082.com:8082/eureka
```


3.服务端启动类加上注解 `@EnableConfigServer`
```java
@EnableEurekaClient
@EnableConfigServer
@SpringBootApplication
public class ConfigServer {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServer.class，args);
    }
}
```

4.启动注册中心，启动服务端，测试一下是否能获取得到配置信息
访问路径 `http://localhost:服务端的端口/你的配置文件-的前部分/你的配置文件-的后半部分{/(这个可选，取那条分支的内容，默认是master)}` 我的访问路径为 `http://localhost:7071/config/dev/master`，如果成功的话，可以看到类似下面的内容，说明你的服务端没问题了
```json
{
  name: "config"，profiles: [
    "dev"
  ]，label: "master"，version: "dcc1f4eb6c3f44c59b28d3699edf39ac4da0e97b"，state: null，propertySources: [
    {
      name: "https://github.com/LCN29/spring-cloud-config/application-config/config-dev.yml"，source: {
        my.name: "dev-update-v1.2"
      }，}
  ]，}
```

5.客户端引入和服务端一样的依赖

6.在resource里面新建一个`bootstrap.yml`的配置文件(启动配置文件，他的优先级高于 application.yml)，在里面配置
```yml
eureka:
  instance:
    instance-id: ${spring.application.name}:${server.port}
  client:
    service-url:
      # 向服务端的注册地址
      defaultZone: http://eureka-8081.com:8081/eureka/，http://eureka-8082.com:8082/eureka

spring:
  cloud:
    config:
      # 下面三个参数对应登录我们服务端获取配置信息的url的三个参数
      name: config
      profile: dev
      label: master
      discovery:
        # Config服务发现支持
        enabled: true
        # config server的应用名
        serviceId: spring-cloud-config-server
```
7.当然客户端还是需要在 `application.yml`设置我们客户端的其他信息的，如端口，应用名
```yml
server:
  port: 9091

spring:
  application:
    name: spring-cloud-config-client
```

8.提供一个方便我们测试的接口
```java
@RestController
public class PropertyController {

    // 我们本地没有配置 my.name 这个 值的，这里的值是从远程拉取过来的
    @Value("${my.name}")
    private String name;

    @GetMapping("/property")
    public String getProperty() {
        String resp = "从远程获取到的配置" + name;
        return resp;
    }
}
```

9.启动客户端，访问 `http://localhost:你的端口/property` 可以看到 `从远程获取到的配置dev`

10.这时候你修改了GitHub上面 config-dev.yml 的内容，你会发现你的客户端没法获取到最新的配置，SpringCloud已经给提供了一个POST方法，请求一下 `/refresh` 这个接口就行了。但是需要做一下配置

11.在客户端引入`actuator`依赖，他里面有我们需要的的 `/refresh`的实现
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

12.在你引用到远程配置的类的上面加上注解`@RefreshScope`，我们这里就是 在我们的的 Controller 上面加上就行了
```java
@RefreshScope
@RestController
public class PropertyController {
}
```

13.`actuator` 默认的 `refresh` 端点是关闭的，我们需要开启(配置在 application.yml 就行了)
```yml
# 暴露所有的端点 management.security.enabled=false 在 spring boot 2.0 版本以上已过期
management:
  endpoints:
    web:
      exposure:
        include: '*'
```

14.当我们的远程仓库的配置继续了修改，只需要我们手动发起一个 Post 请求到 `http://你客户端的地址：端口/actuator/refresh`，你的项目里面的配置就会进行刷新。

15.上面的主动请求 `refresh`，可以通过 GitHub 的 `Webhooks` 进行回调，但是这样，在系统的客户端增加多了，这个样子，我们的 `Webhooks`的url也会增大，不好管理。下面看一下，通过消息总线的方式的解决

## 消息总线

1.流程大概是这样的
>1. 当我们代码提交了，通过GitHub的Webhook功能回调我们的服务端(单机集群都可以)，地址 `http://你的服务端的地址:端口/actuator/bus-refresh`
>2. 这时候服务端通知消息总线(也就是Mq)，消息总线通知所有的客户端
>3. 客户端主动去请求服务端获取最新的配置

2.多配置一个客户端(可参考一下一开始的环境配置)

3.本地搭建一个RabbitMq的环境

4.客户端移除 `actuator`的依赖和配置(这些我们配置在服务端)，同时加上依赖
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-bus</artifactId>
</dependency>
```

5.客户端在bootstrap.yml 里面加一下总线的和rabbitMq的配置
```yml
spring:
  cloud:
    bus:
      # 开启总线
      enabled: true
      trace:
        # 追踪链路
        enabled: true
  rabbitmq:
    host: rabbitMq的地址
    port: rabbitMq的端口
    username: rabbitMq用户名
    password: rabbitMq密码
```

5.服务端加上依赖
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-bus</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

6.在 application.yml 配置文件里面加上配置  
```yml
spring:
  cloud:
    bus:
      # 开启总线
      enabled: true
      trace:
        # 追踪链路
        enabled: true
  rabbitmq:
    host: rabbitMq的地址
    port: rabbitMq的端口
    username: rabbitMq用户名
    password: rabbitMq密码

# 暴露端点
management:
  endpoints:
    web:
      exposure:
        include: bus-refresh    
```

7.在GitHub的webhook 配置上我们的服务端的请求地址 `http://localhost:7071/actuator/bus-refresh`(这个地址，Github是无论都访问不到的，如果是正式环境需要正确的配置)，在这里我们通过模拟发送 Post 请求到 这个地址，达到测试的效果。

8.刷新页面，可以看到你修改并提交到GitHub的内容都能不用重启就取到了

## 代码
[GitHub](https://github.com/LCN29/SpringCloud/tree/master/spring-cloud-config)
