# [SpringCloud] 使用 Nacos 作为注册中心日志不打印


项目使用 log4j2 作为日志框架，配置一切正常，程序启动日志的打印也没有问题，但是引入了 nacos 作为注册中心后，日志在打印了一句 `WARN No Root logger was configured，creating default ERROR-level Root logger with Console appender` 后日志都不打印了。

引入的依赖
```xml 
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <version>0.9.0.RELEASE</version>
</dependency>
```

原因：
在项目的启动过程中，nacos-discovery 会尝试去读系统内的日志配置文件，然后刷新配置，但是如果找不到的话，会使用自身内部的默认配置，但是默认的日志配置没有设置日志的 `Root` 的信息。

*  1. 在 com.alibaba.nacos.client.utils.LogUtils中有
```java 

AbstractNacosLogging nacosLogging
try {
    Class.forName("ch.qos.logback.classic.Logger");
    nacosLogging = new LogbackNacosLogging();
    isLogback = true;
} catch (ClassNotFoundException e) {
    nacosLogging = new Log4J2NacosLogging();
}
nacosLogging.loadConfiguration();

// 从代码可以知道LogUtils，会尝试加载 `ch.qos.logback.classic.Logger` 这个类，加载成功的话，说明项目使用的是 logback 日志框架，加载不到的话，就是使用 log4j2(我的项目中使用的是 log4j2) 
```

* 2. 进入到 `Log4J2NacosLogging` 的构造函数
```java
public class Log4J2NacosLogging extends AbstractNacosLogging {

	public Log4J2NacosLogging() {
		// NACOS_LOG4J2_LOCATION ===> "classpath:nacos-log4j2.xml"
        String location = getLocation(NACOS_LOG4J2_LOCATION);
        // location 有值，将值放到 locationList，用于后续的日志加载
        if (!StringUtils.isBlank(location)) {
            locationList.add(location);
        }
    }

    // 父类的方法，为了方便，写在这里
    protected String getLocation(String defaultLocation) {

    	// 获取系统属性 NACOS_LOGGING_CONFIG_PROPERTY 的值，值不为空 返回 NACOS_LOGGING_CONFIG_PROPERTY 的值
        String location = System.getProperty(NACOS_LOGGING_CONFIG_PROPERTY);
        if (StringUtils.isBlank(location)) {
        	// NACOS_LOGGING_CONFIG_PROPERTY 的值为空，根据是否需要返回默认值，来返回 null 或者  defaultLocation
            if (isDefaultConfigEnabled()) {
            	// 返回 defaultLocation 也就是 classpath:nacos-log4j2.xml，这个配置文件在 nacos-client.jar 这个 jar 包里面可以找到
                return defaultLocation;
            }
            return null;
        }
        return location;
    }

    private boolean isDefaultConfigEnabled() {
    	// 判断是否设置了系统属性 NACOS_LOGGING_DEFAULT_CONFIG_ENABLED_PROPERTY，如果设置了，同时值为 false，则返回false，否则为 true
        String property = System.getProperty(NACOS_LOGGING_DEFAULT_CONFIG_ENABLED_PROPERTY);
        // The default value is true.
        return property == null || BooleanUtils.toBoolean(property);
    }
}
```

* 3. 进入到 `Log4J2NacosLogging` 的 `loadConfiguration` 方法
```java 
public class Log4J2NacosLogging extends AbstractNacosLogging {

	public void loadConfiguration() {
		// 构造函数中处理的 locationList，如果为空，直接返回了
        if (locationList.isEmpty()) {
            return;
        }

        // 获取到当前系统能支持的配置文件的路径
        List<String> configList = findConfig(getCurrentlySupportedConfigLocations());
        if (configList != null) {
            locationList.addAll(configList);
        }

		final List<AbstractConfiguration> configurations = new ArrayList<AbstractConfiguration>();
		LoggerContext loggerContext = (LoggerContext)LogManager.getContext(false);
		for (String location : locationList) {
			try {
				// 把配置文件转为配置
                Configuration configuration = loadConfiguration(loggerContext，location);
                if (configuration instanceof AbstractConfiguration) {
                    configurations.add((AbstractConfiguration)configuration);
                }
            } catch (Exception e) {
                throw new IllegalStateException("Could not initialize Log4J2 Nacos logging from " + location，e);
            }
		}
		CompositeConfiguration compositeConfiguration = new CompositeConfiguration(configurations);
		// 重新加载日志配置
        loggerContext.start(compositeConfiguration);
    }


    private String[] getCurrentlySupportedConfigLocations() {

        List<String> supportedConfigLocations = new ArrayList<String>();

        // YAML_PARSER_CLASS_NAME ===> com.fasterxml.jackson.dataformat.yaml.YAMLParser
        if (ClassUtils.isPresent(YAML_PARSER_CLASS_NAME)) {
            Collections.addAll(supportedConfigLocations，"log4j2.yaml"，"log4j2.yml"，"log4j2-test.yaml"，"log4j2-test.yml");
        }

        // JSON_PARSER_CLASS_NAME ===> com.fasterxml.jackson.databind.ObjectMapper
        if (ClassUtils.isPresent(JSON_PARSER_CLASS_NAME)) {
            Collections.addAll(supportedConfigLocations，"log4j2.json"，"log4j2.jsn"，"log4j2-test.json"，"log4j2-test.jsn");
        }
        supportedConfigLocations.add("log4j2.xml");
        supportedConfigLocations.add("log4j2-test.xml");
        // 把当前系统支持的配置文件的格式，以数组的格式返回回去
        return supportedConfigLocations.toArray(new String[supportedConfigLocations.size()]);
    }


    private List<String> findConfig(String[] locations) {
    	// 把上面 getCurrentlySupportedConfigLocations 获取到的 配置文件格式拼接成程序能找到的路径
        final String configLocationStr = this.strSubstitutor.replace(PropertiesUtil.getProperties().getStringProperty(CONFIGURATION_FILE_PROPERTY));
        if (configLocationStr != null) {
            return Arrays.asList(configLocationStr.split("，"));
        }
        for (String location : locations) {
            ClassLoader defaultClassLoader = ClassUtils.getDefaultClassLoader();
            if (defaultClassLoader != null && defaultClassLoader.getResource(location) != null) {
                List<String> list = new ArrayList<String>();
                list.add("classpath:" + location);
                return list;
            }
        }
        return null;
    }
}
```

从上面看下来，报错的原因是，nacos 会重新加载日志的配置，但是需要的配置文件都没有，后续的默认的配置文件 `classpath:nacos-log4j2.xml` 也没有设置日志的 `Root`，所以报错了。

解决:
1. `isDefaultConfigEnabled` 方法提示了，只需要在项目启动的时候设置一下 `System.setProperty("nacos.logging.default.config.enabled"，"false");` 就行了

2. `getCurrentlySupportedConfigLocations` 方法提示了，只需要把你的日志配置文件的修改为里面的其中的一种格式就行了(因为基于 springboot 搭建的，所以我的日志命名为了 log4j2-spring.xml 所以读取不到)

