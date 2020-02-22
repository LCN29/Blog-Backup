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
>5. 事务的开关，个人的一个习惯，给用户开关，选择是否进行开启功能，默认值 false

那么我们的属性配置类的属性就可以确定为
```java
public class LcnTransactionProperty {

    /** 事务传播行为 */
    private int propagationBehavior = TransactionDefinition.PROPAGATION_REQUIRED;
    
    /** 事务隔离级别*/
    private int isolationLevel = TransactionDefinition.ISOLATION_DEFAULT;
    
    /** 只读事务 */
    private boolean readOnly = false;

    /** 切面表达式 */
    private String advisorExpression;
    
    /** 功能开关 */    
    private boolean enable = false;

    /** 后面省略 get, set */
}
```
从上面知道，我们声明了我们的 starter 能支持的属性，同时属性都赋予了默认值(除了切面表达式, 因为这个不太好做出通用的默认配置，所以没有赋值，所以在后期的时候，会对其进行非空判断，既要求用户必填)

现在我们的属性类完成了一大部分, 接下来我们给我们的属性类，说明我们的 starter 进行配置的 key。
```java
@ConfigurationProperties(prefix = "lcn.transaction")
public class LcnTransactionProperty {
    /** 省略 */
}
```
没错，就是这么简单，在类上加上 `@ConfigurationProperties`, 注解的值就是我们提供给用户的配置 key 的一部分。在使用的时候，注解的配置的值 + 类内部的属性的值，就是完整的配置 key
```yml
lcn:
  transaction:
    enable: true
    advisorExpression: "execution(* com.lcn29.service.impl..*.*(..))"
```
**补充：**  
假设我们的 starter 存在多种功能，比如既提供了事务配置，还提供了权限配置，我们希望配置是下面的格式
```yml
lcn:
  transaction:
    enable: true
  security:
    enable: true    
```
这是我们可以将属性配置类拆分出 3 个类，
>1. 事务的配置类 ---> TransactionProperty;
>2. 权限的配置类 ----> SecurityProperty;
>3. 总的配置类    ----> LcnProperty;

再搭配上 ` @NestedConfigurationProperty` 这个注解就行了

```java
@ConfigurationProperties(prefix = "lcn")
public class LcnProperty {

    @NestedConfigurationProperty
    private TransactionProperty transaction;
    
    @NestedConfigurationProperty
    private SecurityProperty security;

    /** 省略get,set */
}
```
说明：
>1. 我们的真正的配置 TransactionProperty 等, 不需要注解任何注解
>2. 声明在**总的配置类**里面的属性名也是提供给用户配置 key 的一部分

如上面的，TransactionProperty 里面有个属性 `private boolean enable`，那么提供出去的配置 key 为
 **总的配置类 @ConfigurationProperties 的配置值** + **@NestedConfigurationProperty 注解的配置类的属性名** + ** 真正的配置类里面的属性名** , 既最终的配置 key 为 lcn.transaction.enable

## 3. 创建自己的功能类
因为我们的事务 starter 只是一个简单的切面，没有提供出去任何给用户调用的 bean, 所以这里没有。但是为了说明 一个 starter 的流程，我们简单的说一下。  

这里说的功能类，就是我们打算注入到 spring 容器中，用户可以注解使用的，比如我们有一个  TransactionConfigurationReader 的类，可以查看用户配置的属性。
```java
public class TransactionConfigurationReader {

    private String advisorExpression;
    
    public TransactionConfigurationReader(String advisorExpression) {
        this.advisorExpression = advisorExpression;
    }
    
    /** 通过出去的方法 */
    public String getAdvisorExpression() {
        return advisorExpression;
    }
}
```

然后把这个 bean 在我们下面要讲的**配置类**里面进行声明就行了，既
```java
@Bean
public TransactionConfigurationReader transactionConfigurationReader() {
    // lcnTransactionProperty 可以通过 @Resource 注解到 配置类里面的
    return new TransactionConfigurationReader(lcnTransactionProperty.getAdvisorExpression());
} 
```
在引入我们的 starter 的时候， 这个 bean 会自动注入到 Spring 的容器中，用户可以直接使用。

## 3. 创建自己的配置类
(1) 创建自己的配置类
```java
@Aspect
@EnableConfigurationProperties({LcnTransactionProperty.class})
public class TransactionAdviceConfiguration {

    /** 注入我们的属性配置类 */
    @Resource
    private LcnTransactionProperty lcnTransactionProperty;

    /** spring 管理事务的实现类 */
    @Resource
    private DataSourceTransactionManager transactionManager;

    /** 一部分不被允许的事务隔离级别 */
    private final static int[] NO_ALLOW_ISOLATION_LEVEL = {0, 3, 5};

    @Bean
    public Advisor txAdviceAdvisor() {
        AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
        pointcut.setExpression(lcnTransactionProperty.getAdvisorExpression());
        return new DefaultPointcutAdvisor(pointcut, txAdvice());
    }
    
    /**
     * 创建通知
     * @return
     */
    private TransactionInterceptor txAdvice() {

        if (Objects.isNull(lcnTransactionProperty.getAdvisorExpression())) {
            throw new UnsupportedOperationException("the \"lcn.transaction.advisorExpression\" must be set !");
        }

        DefaultTransactionAttribute txAttrRequired = new DefaultTransactionAttribute();
        txAttrRequired.setPropagationBehavior(getPropagationBehavior(lcnTransactionProperty.getPropagationBehavior()));
        txAttrRequired.setIsolationLevel(getIsolationLevel(lcnTransactionProperty.getIsolationLevel()));
        txAttrRequired.setReadOnly(lcnTransactionProperty.getReadOnly());
        NameMatchTransactionAttributeSource source = new NameMatchTransactionAttributeSource();
        source.addTransactionalMethod("*", txAttrRequired);
        return new TransactionInterceptor(transactionManager, source);
    }
    /**
     * 对传播行为的判断
     */
    private int getPropagationBehavior(int curValue) {
        if (curValue >= TransactionDefinition.PROPAGATION_REQUIRED &&
            curValue <= TransactionDefinition.PROPAGATION_NESTED) {
            return curValue;
        }
        return TransactionDefinition.PROPAGATION_REQUIRED;
    }

    /**
     * 对隔离级别的限制
     */
    private int getIsolationLevel(int curValue) {
        for (int value : NO_ALLOW_ISOLATION_LEVEL) {
            if (curValue == value) {
                return TransactionDefinition.ISOLATION_DEFAULT;
            }
        }
        if (curValue >= TransactionDefinition.ISOLATION_DEFAULT &&
            curValue <= TransactionDefinition.ISOLATION_SERIALIZABLE) {
            return curValue;
        }
        return TransactionDefinition.ISOLATION_DEFAULT;
    }
}
```
从上面的配置类，如果去掉了 `@EnableConfigurationProperties` 就是一个简单的切面。  
而 `@EnableConfigurationProperties` 的命名，加上注解的属性值 和我们的配置类可以直接注入我们的属性类，作用应该都知道了吧。
>1. 让 @ConfigurationProperties 注解的类生效，同时注入到容器中
>2. 当 @EnableConfigurationProperties 和 @Configuration 一起使用是， 任何被 @ConfigurationProperties注解的 beans 将自动被 Environment 属性配置。

(2) 让配置类可以注入到 spring 容器中
在项目的 resources 目录下，新建一个 `META-INFO` 的目录，然后在目录里面新建一个文件 `spring.factories`。2个文件名都不能改变，然后在 spring.factories 里面加入
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
上面建的配置类的全路径(例如 com.lcn.starter.transaction.TransactionAdviceConfiguration)
```
(3) 让你的配置类的加载多一下限制
其实到了上面的第 2 步，算是可以用了，但是我们可以让我们配置类，多一些限制，不会导致一引入我们的 starter 就马上创建我们的 bean

首先加上, `@AutoConfigureAfter(DataSourceTransactionManagerAutoConfiguration.class)`
为什么加上这个呢？因为我们的切面需要用到 DataSourceTransactionManager，这个对象的注入是在 
DataSourceTransactionManagerAutoConfiguration 这个配置类里面声明的。所以我们可以让我们的配置类在这个类的初始后

然后加上, `@ConditionalOnBean(DataSourceTransactionManagerAutoConfiguration.class)` 当用户引入了我们的 starter，但是没有引入任何的数据库的配置的话，我们也不希望这个配置类生效

最后，还记得个人的习惯，我们在属性上加上了一个开关，配置的生效的话，我们交给用户去开启，所以我们还需要加上 `@ConditionalOnProperty(value = "lcn.transaction.enable", havingValue = "true")`。当属性值为 true 了才开始初始。

最终我们的配置类的注解有
```java 
@Aspect
@ConditionalOnBean(DataSourceTransactionManagerAutoConfiguration.class)
@AutoConfigureAfter(DataSourceTransactionManagerAutoConfiguration.class)
@ConditionalOnProperty(value = "lcn.transaction.enable", havingValue = "true")
@EnableConfigurationProperties({LcnTransactionProperty.class})
public class TransactionAdviceConfiguration {}
```
至此，我们的配置类就完成了。

## 4. 使用
(1) 第一步：理所当然的引入我们的 starter 依赖咯  
(2) 第二步：确保你的引用里面存在 DataSource。这个正常情况下，在你引入 数据库相关的类都是有的，比如 mybatis 使用了 HikariCP 作为数据源，我们常用的 Druid 也可以
(3) 第三步：开启我们的 starter 的开关，在配置文件加上 lcn.transaction.enable = true
(4) 第四步：当然是开启应用的事务支持，在启动类加上 `@EnableTransactionManagement`