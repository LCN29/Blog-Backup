作者：Derobukal
链接：https://www.nosuchfield.com/2017/10/15/Spring-Boot-Starters/
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

# 【Spring】Spring Boot Starter
Spring Boot Starter 是在 SpringBoot 组件中被提出来的一种概念，stackoverflow 上面已经有人概括了这个starter 是什么东西，想看完整的回答戳 [这里](https://stackoverflow.com/a/28273660)

`Starter POMs are a set of convenient dependency descriptors that you can include in your application. You get a one-stop-shop for all the Spring and related technology that you need, without having to hunt through sample code and copy paste loads of dependency descriptors. For example, if you want to get started using Spring and JPA for database access, just include the spring-boot-starter-data-jpa dependency in your project, and you are good to go.`

大概意思就是说 starter 是一种对依赖的 synthesize（合成），这是什么意思呢？我可以举个例子来说明。

## 传统的做法
在没有 starter 之前，假如我想要在 Spring 中使用 jpa，那我可能需要做以下操作：
>1. 在 Maven 中引入使用的数据库的依赖（即 JDBC 的 jar）
>2. 引入 jpa 的依赖
>3. 在 xxx.xml 中配置一些属性信息
>4. 反复的调试直到可以正常运行

需要注意的是，这里操作在我们**每次新建一个需要用到 jpa 的项目的时候都需要重复的做一次**。也许你在第一次自己建立项目的时候是在 Google 上自己搜索了一番，花了半天时间解决掉了各种奇怪的问题之后，jpa 终于能正常运行了。有些有经验的人会在 OneNote 上面把这次建立项目的过程给记录下来，包括操作的步骤以及需要用到的配置文件的内容，在下一次再创建 jpa 项目的时候，就不需要再次去 Google 了，只需要照着笔记来，之后再把所有的配置文件 copy&paste 就可以了。

像上面这样的操作也不算不行，事实上我们在没有 starter 之前都是这么干的，但是这样做有几个问题：
>1. 如果过程比较繁琐，这样一步步操作会增加出错的可能性
>2. 不停地 copy&paste 不符合 [Don’t repeat yourself](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) 精神
>3. 在第一次配置的时候（尤其如果开发者比较小白），需要花费掉大量的时间

## 使用 Spring Boot Starter 提升效率
starter 的主要目的就是为了解决上面的这些问题。

starter 的理念：starter 会把所有用到的依赖都给包含进来，避免了开发者自己去引入依赖所带来的麻烦。需要注意的是不同的 starter 是为了解决不同的依赖，所以它们内部的实现可能会有很大的差异，例如 jpa 的starter 和 Redis 的 starter 可能实现就不一样，这是因为 starter 的本质在于 synthesize，这是一层在逻辑层面的抽象，也许这种理念有点类似于 Docker，因为它们都是在做一个“包装”的操作，如果你知道 Docker 是为了解决什么问题的，也许你可以用 Docker 和 starter 做一个类比。

starter 的实现：虽然不同的 starter 实现起来各有差异，但是他们基本上都会使用到两个相同的内容：ConfigurationProperties 和 AutoConfiguration。因为 Spring Boot 坚信“约定大于配置”这一理念，所以我们使用 ConfigurationProperties 来保存我们的配置，并且这些配置都可以有一个默认值，即在我们没有主动覆写原始配置的情况下，默认值就会生效，这在很多情况下是非常有用的。除此之外，starter 的ConfigurationProperties 还使得所有的配置属性被聚集到一个文件中（一般在 resources 目录下的application.properties），这样我们就告别了 Spring 项目中 XML 地狱。

starter 的整体逻辑：
![Alt '图片'](https://imgconvert.csdnimg.cn/aHR0cHM6Ly93d3cubm9zdWNoZmllbGQuY29tL2ltYWdlcy8yMDE3MTAxNS8yMDE3MTAxNTEyMTg0OC5wbmc?x-oss-process=image/format,png#pic_center)
上面的 starter 依赖的 jar 和我们自己手动配置的时候依赖的 jar 并没有什么不同，所以**我们可以认为 starter 其实是把这一些繁琐的配置操作交给了自己，而把简单交给了用户**。除了帮助用户去除了繁琐的构建操作，在“约定大于配置”的理念下，ConfigurationProperties 还帮助用户减少了无谓的配置操作。并且因为 application.properties 文件的存在，即使需要自定义配置，所有的配置也只需要在一个文件中进行，使用起来非常方便。

了解了 starter 其实就是帮助用户简化了配置的操作之后，要理解 starter 和被配置了 starter 的组件之间并不是竞争关系，而是辅助关系，即我们可以给一个组件创建一个 starter 来让最终用户在使用这个组件的时候更加的简单方便。基于这种理念，我们可以给任意一个现有的组件创建一个 starter 来让别人在使用这个组件的时候更加的简单方便，事实上 Spring Boot 团队已经帮助现有大部分的流行的组件创建好了它们的 starter，你可以在这里查看这些 starter 的列表。

## 创建自己的 Spring Boot Starter
如果你想要自己创建一个 starter，那么基本上包含以下几步
>1. 创建一个 starter 项目，关于项目的命名你可以参考[这里](https://docs.spring.io/spring-boot/docs/2.0.0.M5/reference/htmlsingle/#boot-features-custom-starter-naming)
>2. 创建一个 ConfigurationProperties 用于保存你的配置信息（如果你的项目不使用配置信息则可以跳过这一步，不过这种情况非常少见）
>3. 创建一个 AutoConfiguration，引用定义好的配置信息；在 AutoConfiguration 中实现所有 starter 应该完成的操作，并且把这个类加入 spring.factories 配置文件中进行声明
>4. 打包项目，之后在一个 Spring Boot 项目中引入该项目依赖，然后就可以使用该 starter 了

我们来看一个例子（例子的完整代码位于https://github.com/RitterHou/learn-spring-boot-starter）

首先新建一个Maven项目，设置 pom.xml 文件如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <artifactId>http-starter</artifactId>
    <version>0.0.1-SNAPSHOT</version>

    <!-- 自定义starter都应该继承自该依赖 -->
    <!-- 如果自定义starter本身需要继承其它的依赖，可以参考 https://stackoverflow.com/a/21318359 解决 -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starters</artifactId>
        <version>1.5.2.RELEASE</version>
    </parent>

    <dependencies>
        <!-- 自定义starter依赖此jar包 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>
        <!-- lombok用于自动生成get、set方法 -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.16.10</version>
        </dependency>
    </dependencies>

</project>
```

创建proterties类来保存配置信息：
```java
@ConfigurationProperties(prefix = "http") // 自动获取配置文件中前缀为http的属性，把值传入对象参数
@Setter
@Getter
public class HttpProperties {

    // 如果配置文件中配置了http.url属性，则该默认属性会被覆盖
    private String url = "http://www.baidu.com/";

}
```

上面这个类就是定义了一个属性，其默认值是 http://www.baidu.com/，我们可以通过在 application.properties 中添加配置 http.url=https://www.zhihu.com 来覆盖参数的值。

创建业务类：
```java
@Setter
@Getter
public class HttpClient {

    private String url;

    // 根据url获取网页数据
    public String getHtml() {
        try {
            URL url = new URL(this.url);
            URLConnection urlConnection = url.openConnection();
            BufferedReader br = new BufferedReader(new InputStreamReader(urlConnection.getInputStream(), "utf-8"));
            String line = null;
            StringBuilder sb = new StringBuilder();
            while ((line = br.readLine()) != null) {
                sb.append(line).append("\n");
            }
            return sb.toString();
        } catch (Exception e) {
            e.printStackTrace();
        }
        return "error";
    }

}
```

这个业务类的操作非常简单，只包含了一个 url 属性和一个 getHtml 方法，用于获取一个网页的 HTML 数据，读者看看就懂了。

创建 AutoConfiguration
```java
@Configuration
@EnableConfigurationProperties(HttpProperties.class)
public class HttpAutoConfiguration {

    @Resource
    private HttpProperties properties; // 使用配置

    // 在Spring上下文中创建一个对象
    @Bean
    @ConditionalOnMissingBean
    public HttpClient init() {
        HttpClient client = new HttpClient();

        String url = properties.getUrl();
        client.setUrl(url);
        return client;
    }

}
```

在上面的 AutoConfiguration 中我们实现了自己要求：在 Spring 的上下文中创建了一个 HttpClient 类的 bean，并且我们把 properties 中的一个参数赋给了该 bean。
关于 @ConditionalOnMissingBean 这个注解，它的意思是在该 bean 不存在的情况下此方法才会执行，这个相当于开关的角色，更多关于开关系列的注解可以参考这里。

最后，我们在 resources 文件夹下新建目录 META-INF，在目录中新建 spring.factories 文件，并且在 spring.factories 中配置 AutoConfiguration：
```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.nosuchfield.httpstarter.HttpAutoConfiguration
```

到此，我们的 starter 已经创建完毕了，使用 Maven 打包该项目。之后创建一个 Spring Boot 项目，在项目中添加我们之前打包的 starter 作为依赖，然后使用 Sring Boot 来运行我们的 starter，代码如下：
```java
@Component
public class RunIt {

    @Resource
    private HttpClient httpClient;

    public void hello() {
        System.out.println(httpClient.getHtml());
    }
}
```

正常情况下此方法的执行会打印出 url http://www.baidu.com/ 的 HTML 内容，之后我们在application.properties 中加入配置：
```
http.url=https://www.zhihu.com/
```

再次运行程序，此时打印的结果应该是知乎首页的 HTML 了，证明 properties 中的数据确实被覆盖了。