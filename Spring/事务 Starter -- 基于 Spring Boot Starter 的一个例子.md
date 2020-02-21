# [Spring] 事务 Starter -- 基于 Spring Boot Starter 的一个例子
在开始之前, 如果你还不知道什么是 Spring Boot Starter 的话，可以看一下这篇文章。

## 概述
在 Spring 中如果要使用事务的话，可以说是有 2 种方式
>1.  使用 `@Transactional` 注解，被其修饰的类的所有方法都具有事务，或者其注解的方法，具有事务。
>2. 通过切面，配合 `DataSourceTransactionManager` 进行配置

Spring 并不直接管理事务，而是提供了多种事务管理器, 将事务管理的职责委托给 Hibernate 或者 JTA 等持久化机制所提供的相关平台框架的事务来实现。spring 只提供了对应的接口 `PlatformTransactionManager `, 不同的框架会有不同的实现
| 实现类 | 说明|
| :-: | :-: |
| DataSourceTransactionManager | 使用 Spring JDBC 或者 iBatis 进行持久化时使用 |
| HibernateTransactionManager | 使用 Hibernate 进行持久化时使用 |
| JpaTransactionManager  | 使用 Jpa 进行持久化时使用  |
| ...| ...|

因为在实际中常用到的可能是 JDBC 和 MyBaits， 所以使用了 DataSourceTransactionManager 进行演示。
将其封装成为一个 starter，做到开箱即用。那么开始吧 ！

## 1.  引入相关的依赖
```xml
 <dependencies>
 
        <!-- 需要 spring boot 的一些特性 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
            <optional>true</optional>
        </dependency>

		<!-- 用于解决 Spring Boot Configuration Annotation Processor not found in classpath 这个提示 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- 基于 aop 实现 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter</artifactId>
                </exclusion>
            </exclusions>
        </dependency>

        <!--  数据库事务需要的依赖 -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jdbc</artifactId>
        </dependency>
</dependencies>
```

## 2. 创建一个属性类
在我们使用 Spring Boot 的时候，我们可以在配置文件 `application.yml` 或者 `application.properties ` 进行配置，修改程序的一些默认配置，比如应用启动的端口
```yml
server:
  port: 12345
```
默认端口是 8080， 通过这个配置可以任意低修改我们的配置端口。这里体现了 Spring Boot 常说的 "约定大于配置" 的特性。提供了默认的配置，但是也允许你进行修改。我们这里的属性类，就是为了实现上面的功能，将 Starter 需要的配置进行一些默认的配置，同时提供一个属性 key 给用户进行修改, 如上面的 `server.port`。 

回到我们的 Starter 这里, 分析一下我们的 Starter 需要提供哪些配置。因为我们的 Starter 是基于 DataSourceTransactionManager 实现的，所以我们可以从他的提供的配置，可以分析出
>1. 事务的传播行为，默认的值为 0 (如果当前没有事务，就新建一个事务，如果已经存在一个事务中，加入到这个事务中)
>2. 事务的隔离级别，默认值为 -1(Default, 使用数据库默认的事务隔离级别)
>3. 事务是否是只读事务，默认值为 false
>4. 事务起作用的切面表达式
>5. 事务的开关，个人的一个习惯，给用户一个开关，进行配置

那么我们的属性配置类的属性就可以确定为
```java
public class LcnTransactionProperty {

	private int propagationBehavior = TransactionDefinition.PROPAGATION_REQUIRED;

	private int isolationLevel = TransactionDefinition.ISOLATION_DEFAULT;

	private boolean readOnly = false;

}
```