# [SpringCloud] Hystrix 简单使用

## 前提说明
因为 SpirngBoot，SpringCloud 的各个版本之间差异还是挺大的，所以在参照本博客进行学习时，有可能出现因为版本不一致，而出现不同的问题。如果可以和本项目使用的环境保持一致，即使不一致，也尽可能不要跨大版本。
>1. jdk8
>2. SpringBoot : 2.1.4.RELEASE
>3. SpringCloud : Greenwich.SR1
>4. Maven 多模块

## 准备
参考一下 [这个](https://blog.csdn.net/LCN29/article/details/102087449) 里面的环境准备，搭建出2个服务端，1个服务提供者，1个服务调用者

其中服务提供者的 Rest API 可有是这样的
```java
@RestController
public class ProviderController {

    @GetMapping("/hello/{delay}")
    public String hello(@PathVariable("delay")int delay，HttpServletRequest request) {
        // 传递的参数为1，让线程睡5s，让调用方超时
        if (delay == 1) {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        //打印请求的地址
        String resp = " 当前响应来自于" + request.getRequestURL();
        return resp;
    }
}
```

## Hystrix 开始

* 在服务调用者的依赖里面加入 Hystrix 的依赖

```xml
<!-- 熔断器 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

* 启动类加上 `@EnableCircuitBreaker` 注解，启动熔断功能

```java
@EnableFeignClients
@EnableEurekaClient
@EnableCircuitBreaker
@SpringBootApplication
public class ClientConsumer {

    public static void main(String[] args) {
        SpringApplication.run(ClientConsumer.class，args);
    }
}
```

* 按照 Fetch 新建一个调用相关的接口 和 服务调用方提供给浏览器调用的接口

```java
@FeignClient(name = "client-provider")
public interface ProviderRemote {

   /**
    * 远程接口
    * @param delay 是否进入延迟 1：开启
    * @return
    */
   @GetMapping("/hello/{delay}")
   String hystrix(@PathVariable("delay")int delay);
}


/**
 * 供浏览器调用的接口
 */
@RestController
public class HystrixController {

    @Resource
    private ProviderRemote providerRemote;

    @GetMapping("/hystrix/{delay}")
    public String hystrix(@PathVariable("delay")int delay) {
        String resp = providerRemote.hystrix(delay);
        return resp;
    }
}
```
到现在为止，我们的服务调用类，还是没有熔断的功能的

* 加上熔断的功能  

为我们的 调用相关的接口 创建一个实现类，并注入容器，同时设置 @FeignClient 的 fallback 选项为我们的实现类
```java
@Component
public class ProviderRemoteImpl implements ProviderRemote {
    @Override
    public String hystrix(int delay) {
        return "服务提供方出现异常，不进行调用直接返回了";
    }
}


@FeignClient(name = "client-provider"，fallback = ProviderRemoteImpl.class)
public interface ProviderRemote {

   /**
    * 远程接口
    * @param delay 是否进入延迟 1：开启
    * @return
    */
   @GetMapping("/hello/{delay}")
   String hystrix(@PathVariable("delay")int delay);
}
```

* 最后开启熔断器的功能

```yml
feign:
  hystrix:
    # 开启熔断器功能
    enabled: true
```

## 通过 Hystrix-dashboard 进行监控
Hystrix 提供了一套实时监控的工具，通过 HystrixDashboard 我们可以在直观地看到各 Hystrix Command 的请求响应时间，请求成功率等数据。下面就介绍一下怎么使用

* 需要我们的 Hystrix 项目提供一个注册一个Servlet，基于监控项目使用，也就是在我们的服务调用者那边添加

```java
@Configuration
public class MetricsStreamServletConfigration {

    @Bean
    public ServletRegistrationBean getServlet(){
        HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
        ServletRegistrationBean registrationBean = new ServletRegistrationBean(streamServlet);
        registrationBean.setLoadOnStartup(1);
        registrationBean.addUrlMappings("/actuator/hystrix.stream");
        registrationBean.setName("HystrixMetricsStreamServlet");
        return registrationBean;
    }
}
```

* 新建一个监控项目，项目需要3个依赖 hystrix，hystrix-dashboard 和 actuator

```xml
<!-- hystrix 监控 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
    <version>1.4.7.RELEASE</version>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

* 在启动类添加注解 `@EnableHystrixDashboard`

```java
@SpringBootApplication
@EnableHystrixDashboard
public class MonitorDashBoard {
    public static void main(String[] args) {
        SpringApplication.run(MonitorDashBoard.class，args);
    }
}
```

* 启动项目后，在浏览器输入 `http://localhost:你设置的端口/hystrix`可以看到下面的界面, 在第一个空格输入你的要监控的 hystrix 项目，这里就是服务调用者的地址 `http://localhost:端口/actuator/hystrix.stream`，下面的Title 随意。
![Alt 'hystrix-dashboard'](https://s2.ax1x.com/2019/10/05/uy3uAf.png)


* 输入后打开，可以看到这个界面 (如果界面一直在 loading，手动调用一次 服务调用者的接口就可以了)
![Alt 'hystrix-dashboard'](https://s2.ax1x.com/2019/10/05/uy1sTf.png)


* dash-board 可以查看每个应用的信息，但是每次都只能查看一个，有时我们需要了解这个集群的情况，dash-board 就满足不了了，这时可以使用 `Turbine`，这里就不做更多的说明了。

## 代码
[GitHub](https://github.com/LCN29/SpringCloud/tree/master/spring-cloud-hystrix)
