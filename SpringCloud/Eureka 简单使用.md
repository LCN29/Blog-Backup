
# [SpringCloud] Eureka 简单使用

## 前提说明
因为 SpirngBoot，SpringCloud 的各个版本之间差异还是挺大的，所以在参照本博客进行学习时，有可能出现因为版本不一致，而出现不同的问题。如果可以和本项目使用的环境保持一致，即使不一致，也尽可能不要跨大版本。
>1. jdk8
>2. SpringBoot : 2.1.4.RELEASE
>3. SpringCloud : Greenwich.SR1
>4. Maven 多模块

## 准备
1. 先建立一个 maven 的父模块，也就是整个项目里面只有一个 pom 文件
2. 在父 pom 里面添加一些 共用的配置，比如 SpringBoot，SpringCloud 的依赖，可以参考这一个:  [Pom](https://github.com/LCN29/SpringCloud/blob/master/spring-cloud-eureka/pom.xml "父pom配置")


## 建立一个单机的 Eureka 服务端
1 在父模块里面建立一个子模块，我的模块名叫做：`eureka-server-one`  
2 在子模块引入 Eureka 服务端需要的依赖
```xml
<dependencies>
    <!-- eureka 服务端的依赖-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
```

3 然后在启动类上加上注解 `@EnableEurekaServer`
```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerOne {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerOne.class，args);
    }
}
```

4 项目一直在报错，Eureka 会把自己当做一个服务注册到服务端，此处需要停止他的这个行为，在 SpringBoot 的配置文件 `application.yml` (maven模块是没有这个文件的，需要在 src/main/ 下面建立一个 resources 文件夹，然后在里面收到创建这个文件)，加上这2个配置
```yml
# 应用的名字
spring:
  application:
    name: eureka-server
eureka:
  client:
    # 禁止把自己注册到 eureka 的服务端
    register-with-eureka: false
    # 不从 eureka 服务端拉取节点信息
    fetch-registry: false
    # 我们知道 eureka 也会把自己当做服务进行注册，但是注册的地址? 下面配置的就是注册的地址，默认为 http://localhost:8761/eureka/
    service-url:
      defaultZone: http://localhost:8080/eureka/
```
5 通过浏览器访问 `http://localhost:8080/`，就能看到 eureka 的管理页面  

6 这时候虽然程序运行起来的，但是如果你查看页面的 `General Info` 项里面的 `unavailable-replicas` 会发现我们的 eureka 服务端是显示为不可用的，但是他是可用的。

自此，我们的 Eureka 的服务端就可以了。

## 建立一个 Eureka 客户端
1 在父模块里面建立一个子模块，我的模块名叫做：`eureka-client-one`  
2 在子模块引入 Eureka 客户端需要的依赖
```xml
<dependencies>
    <!-- eureka 服务端的依赖-->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>

    <!-- web 功能的支持，没有这个 客户端就会启动完就结束程序 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

3 然后在启动类上加上注解 `@EnableEurekaClient`
```java
@EnableEurekaClient
@SpringBootApplication
public class EurekaClientOne {
    public static void main(String[] args){
        SpringApplication.run(EurekaClientOne.class，args);
    }
}
```

4 添加配置
```yml
spring:
  application:
    name: eureka-client-one
server:
  port: 9091

eureka:
  client:
    service-url:
      # 服务端的注册地址
      defaultZone: http://localhost:8080/eureka/
```
5 打开 eureka 的服务端界面，可以看到 `Instances currently registered with Eureka` 项里面有刚刚启动的客户端的ApplicationName，同时状态为 UP。

## Eureka 客户端间服务调用
1 在父模块里面建立一个子模块，我的模块名叫做：`eureka-client-two`，作为服务的调用方

2 依赖，启动类，配置 和 `eureka-client-one` 一样，当然记得把 `application.yml` 中的 applicationName 修改为 `eureka-client-two`，端口修改为另一个没有使用的，这样我们就有2个客户端了  

3 现在我们让 `eureka-client-one` 作为服务的提供方，提供一个 Rest API 接口
```java
@RestController
public class MyServerController {

    @GetMapping("/server/{id}")
    public String server(@PathVariable("id") int id) {
        return "收到请求Id : " + id + "，结束";
    }
}
```

4 服务调用方 `eureka-client-two` 引入 Feign，作为服务的方式
```xml
<!--远程调用-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```
在 spring-mvc 里面有一个 `RestTemplate` 的功能可以进行远程调用，但是那个有些麻烦，所以有了基于 `RestTemplate` 二次开发的 Feign。

5 Feign 使用第一步，创建一个接口
```java

// name 是服务提供方的应用名
@FeignClient(name = "eureka-client-one")
public interface RemoteServer {

    // 调用的 Rest API，同时参数需要和远程的 Rest API 一样
    @GetMapping("/server/{id}")
    String server(@PathVariable("id")int id);
}
```

6 Feign 使用第二步，注入我们的接口
```java
@RestController
public class FeignController {

    @Resource
    private RemoteServer remoteServer;

    @GetMapping("/feign/{id}")
    public String feign(@PathVariable("id")int id) {
        String result = remoteServer.server(id);
        return result;
    }
}
```

7 Feign 使用第三步，然后在启动类上加上注解 `@EnableFeignClients`
```java
@SpringBootApplication
@EnableEurekaClient
@EnableFeignClients
public class EurekaClientTwo {
    public static void main(String[] args){
        SpringApplication.run(EurekaClientTwo.class，args);
    }
}
```

8 依次启动 `eureka-server`，`eureka-client-one`，`eureka-client-two`，然后在浏览器输入`http://localhost:{eureka-client-two设置的端口}/feign/123`，可以看到`收到请求Id : 123，结束`就是成功了

## Eureka 服务端集群配置
我们知道 Eureka 作为注册中心，一旦挂了，基本整个系统就可能无法使用了(如果Eureka是在运行过一段时间后才挂的，同时各个客户端之间都有其他客户端的缓存，还是能通信的，就是无法加入新节点)，所以无特殊情况，Eureka 服务端都是以集群的形式部署的

1 同样的依照 `eureka-server-one`，新建一个 `eureka-server-two`，启动类，配置，依赖都一样。

2 修改 `eureka-server-one` 和  `eureka-server-two` 的配置
```java
# 这个是 eureka-server-one 的配置
spring:
  application:
    name: eureka-server
server:
  port: 8081
eureka:
  client:
    service-url:
      # 向另外一个服务端注册自己，如果有多个服务端，通过逗号分隔就行了
      defaultZone: http://localhost:8082/eureka/


# 这个是 eureka-server-two 的配置
spring:
  application:
    name: eureka-server
server:
  port: 8082
eureka:
  client:
    service-url:
      # 向另外一个服务端注册自己，如果有多个服务端，通过逗号分隔就行了
      defaultZone: http://localhost:8081/eureka/      
```
首先因为是集群配置，所以2个服务端的应用名都是一样的

3 启动项目(第一个启动的会报错，因为找不到需要的注册中心，当你把第二个注册中心启动了，就不会报错了)依次访问 `http://localhost:8081` 和 `http://localhost:8082` 都可以访问，但是你会发现你的服务端都是 `unavailable-replicas`的。这是因为： `eureka.client.serviceUrl.defaultZone配置项的地址，不能使用localhost，要使用域名`

4 在你电脑的 hosts 文件里面添加这2行
```txt
127.0.0.1 eureka-8081.com
127.0.0.1 eureka-8082.com
```
5 修改配置文件的 `defaultZone` 项，同时增加 `Instances`的配置，其他的都不用修改
```yml
# 这个是 eureka-server-one 的配置
eureka:
  instance:
    # 当前服务的域名
    hostname: eureka-8081.com
    # 实例的名字，上面的applicationName可以看出一个组，而这里是说明当前的服务是组中的哪一个
    instance-id: ${spring.application.name}:${server.port}
  client:
    service-url:
      # 向另外一个服务端注册自己，如果有多个服务端，通过逗号分隔就行了
      defaultZone: http://eureka-8082.com:8082/eureka/

# 这个是 eureka-server-two 的配置
eureka:
  instance:
    # 当前服务的域名
    hostname: eureka-8082.com
    # 实例的名字，上面的applicationName可以看出一个组，而这里是说明当前的服务是组中的哪一个
    instance-id: ${spring.application.name}:${server.port}
  client:
    service-url:
      # 向另外一个服务端注册自己，如果有多个服务端，通过逗号分隔就行了
      defaultZone: http://eureka-8081.com:8082/eureka/
```

6 最终的结果
![Alt '集群结果'](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9zMi5heDF4LmNvbS8yMDE5LzEwLzAzL3UwV2FDOS5wbmc?x-oss-process=image/format，png)


## 代码
[GitHub](https://github.com/LCN29/SpringCloud/tree/master/spring-cloud-eureka)
