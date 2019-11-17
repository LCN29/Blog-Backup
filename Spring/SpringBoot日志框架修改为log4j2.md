# [Spring]SpringBoot 日志框架修改为 log4j2

在使用 SpringBoot 的时候，默认使用的日志框架是 logback。 现在如果想要使用 log4j2，如何做修改？


>1. 排除 SpringBoot 默认的 logback 依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

>2. 添加 log4j2 的依赖
```xml
<!--  使用 log4j2 替代 logback      -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

>3. 在 resources 目录下新建一个 log4j2-spring.xml 的日志配置文件

一般情况下，这样就能在 SpringBoot 里面使用 log4j2 了。 但是还能做继续做一些完善的！


* 完善1: 你项目启动的时候，可能会看到这样的日志信息
```
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/E:/Repertory/ch/qos/logback/logback-classic/1.2.3/logback-classic-1.2.3.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/E:/Repertory/org/apache/logging/log4j/log4j-slf4j-impl/2.12.1/log4j-slf4j-impl-2.12.1.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [ch.qos.logback.classic.util.ContextSelectorStaticBinder]
```

这个虽然不影响使用，但是强迫症，看着不舒服。 至于原因：就是项目里面使用了 slf4f 做门面，但是项目里面有复数的实现方式，解决方式也很简单：将项目里面的 其他的日志框架排除就行了。 比如使用 myBatis 导入的 starter 里面就有 logbak 的实现.
```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>${mybatis.starter.version}</version>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```
项目的依赖查看，可以通过 Idea 右侧的 Maven 选项进行查看等。


* 完善2： 使用彩色日志

在使用 SpringBoot 默认的日志框架时，可以发现输出在控制台的日志是彩色的，但是现在我们换成了 log4j2，日志却是白色的，其实这里面也是支持配置的。
首先配置控制台的输出格式：
```xml
<Property name ="ConsoleLogPattern">%d{yyy-MM-dd HH:mm:ss} %style{[%15t]}{bright，blue} %clr{%-5level} %style{%logger{80}}{cyan} %style{[%L]}{magenta} - %msg%n</Property>
```
格式基本就是 `%style{日志}{想要显示的颜色}`  

然后引入到控制台输出，然后设置 `disableAnsi="false" noConsoleNoAnsi="false"` 就能达到彩色日志的效果
```xml
<Console name="console" target="SYSTEM_OUT">
    <!--输出日志的格式-->
    <patternLayout pattern="${ConsoleLogPattern}" disableAnsi="false" noConsoleNoAnsi="false"/>
</Console>
```

* 完善3： 在 spring-log4j2.xml 使用 application.yml 配置的属性

在 SpringBoot + logback 的时候，我们可以通过在 application.yml 里面配置 `logging.file.path=日志路径` 来指定日志的保存地方，但是在使用 log4j2 的时候，不起作用了。 原因是： log4j2-spring.xml 是不归 spring 管理的，所以也就没法读取到 application.yml 里面的配置了。 解决方式： 通过 spring 的 监听器(Listener)功能，将我们读取到的 application.yml 的日志路径设置到系统属性，然后在日志文件里面读取对应的系统属性就行了。

首先：新建一个监听器
```java
public class LoggingListener implements ApplicationListener，Ordered {

    /**
     * 提供给日志文件读取配置的key，使用时需要在前面加上 sys:
     */
    private final static String LOG_PATH = "log.path";

    /**
     * spring 内部设置的日志文件的配置key
     */
    private final static String SPRING_LOG_PATH_PROP = "logging.file.path";

    @Override
    public void onApplicationEvent(ApplicationEvent applicationEvent) {

        if (applicationEvent instanceof ApplicationEnvironmentPreparedEvent) {
            ConfigurableEnvironment environment = ((ApplicationEnvironmentPreparedEvent) applicationEvent).getEnvironment();
            String filePath = environment.getProperty(SPRING_LOG_PATH_PROP);
            if (filePath != null && !filePath.isBlank()) {
                System.setProperty(LOG_PATH，filePath);
            }
        }
    }

    @Override
    public int getOrder() {
    	// 当前监听器的启动顺序需要在日志配置监听器的前面，所以此处减 1
        return LoggingApplicationListener.DEFAULT_ORDER - 1;
    }
}
```

然后: 在项目启动的时候，将我们的监听器交给应用
```java
@SpringBootApplication
public class ApplicationStarter {

    public static void main(String[] args) {

        SpringApplication application = new SpringApplication(ApplicationStarter.class);
        // 添加 日志监听器，使 log4j2-spring.xml 可以间接读取到配置文件的属性
        application.addListeners(new LoggingListener());
        application.run(args);
    }
}
```

最后: log4j2-spring.xml 里面的日志路径配置
```xml
<!--  文件保存路径 前缀 sys: 不能省   -->
<Property name="LogPath">${sys:log.path}</Property>
```

至此： SpringBoot 整合 log4j2 就此结束了！

## 参考
[Use Spring boot application properties in log4j2.xml](https://stackoverflow.com/questions/48941104/use-spring-boot-application-properties-in-log4j2-xml)
