

## [Java基础] Jdk8新特性

### 1. 语言的新特性

1.1 接口里面可以有具体的实现逻辑。 通过使用关键字 `default` 修饰的方法，可以在接口内实现具体的逻辑(实现类可以重写这个方法)
```java 
public interface MyInterface {

    default String sayHello() {
        return "Hello";
    }
}

public class MyInterfaceImpl implements MyInterface {

    @Override
    public String sayHello() {
        return "Hi";
    }
}
```

1.2 接口里面可以有静态方法。 和 default 方法类似，但是接口的静态方法没法被实现类重写(实现类可以声明 同名同参数同返回值的静态方法，但是这个方法没法被 `@Override` 修饰，同时方法的修饰符可以比接口内的广，所以不是重写)
```java
public interface MyInterface {

    private static void method() {
        System.out.println("static method");
    }
}

public class MyInterfaceImpl implements MyInterface {

    public static void method() {
        System.out.println("static method");
    }
}
```

1.3 Lambda 表达式  
lambda 的介绍，可以看[这里](https://blog.csdn.net/LCN29/article/details/103080361)。

1.4 函数式接口  
一个接口除了 default 方法 和 static 方法外，只有一个需要实现的方法，同时接口被 @FunctionalInterface 标记。
```java 
@FunctionalInterface
public interface Runnable {

    public abstract void run();
}
```

新增的四个重要的函数式接口：函数形接口 、供给形接口、消费型接口、判断型接口(还要其他的，其他的大概都是前面四个的变种)
```java 
@FunctionalInterface
public interface Function<T，R> {
	/**
	 * T 作为输入，返回的 R类型 作为输出
	 */
	R apply(T t);
}

@FunctionalInterface
public interface Supplier<T> {

    /**
     * 没有输入 ，R 作为输出
     */
    T get();
}

@FunctionalInterface
public interface Consumer<T> {

    /**
     * T 作为输入 ，没有输出
     */
    void accept(T t);
}

@FunctionalInterface
public interface Predicate<T> {

	/**
	 * T 作为输入 ，返回 boolean 值的输出
	 */
    boolean test(T t);
}
```

1.5 新的方法引用(一般需要配合 lambda 表达式使用)

* 构造方法引用  `Obj ::new ` (需要对应的对象有一个无参的构造函数)

```java

public String getString(Supplier<String> supplier) {
	return supplier.get();
}

/* 调用 相当于  getString(() -> new String())  => 
 *
 *  getString(new Supplier<String>() { 
 *		@Override
 *		public String get() { 
 *			return new String(); 
 *		}  
 *	});
 *
 */
getString(String::new);
```

* 静态方法引用  `obj :: static_method `
```java 
public class Main {

	public static void fn(String str) {
        System.out.println(str);
    }
}

List<String> list = new ArrayList<>();
list.forEach(Main::fn);
```

* 实例对象方法引用 `obj_instance :: method `
```java 
public class Main {

	public void fn(String str) {
        System.out.println(str);
    }
}

Main main = new Main();
List<String> list = new ArrayList<>();
list.forEach(main::fn);
```

1.6 重复注解  
声明的注解通过 `@Repeatable` 修饰的时候，这个注解可以重复注解在同一地方
```java 
public @interface MyAnnotations {
    MyAnnotation[] value();
}

/**
 * 后续 MyAnnotation 就可以重复注解
 */
@Repeatable(MyAnnotations.class)
public @interface MyAnnotation {
    String value() default "";
}
```

1.7 扩展注解的支持  
更多的注解的地方： 局部变量，接口类型，方法参数等。

1.8 更强大的类型推导


### 2. Java 官方库的新特性

2.1 Stream
Stream 的介绍，可以看[这里](https://blog.csdn.net/LCN29/article/details/103098279)


2.2 Optional

2.3 Date/Time API 新的时间类

2.4 Base64  base64 的实现不在通过原生实现

2.5 JUC 工具包扩充


### 3. Java 编译器的新特性
在class文件中保留参数名

```java 
public class TestMethodParams {

	public String test(String name，int age) {
        return name;
    }

    public static void main(String[] args) throws NoSuchMethodException，SecurityException {

		Method method = TestMethodParams.class.getMethod("test"，String.class，int.class);
		for (Parameter p : method.getParameters()) {
			System.out.println("parameter: " + p.getType().getName() + "，" + p.getName());
		}
    }
}
```

不做任何处理的结果为:
```
parameter: java.lang.String，arg0
parameter: int，arg1
```

开启新特性，需要在 Java 编译的时候，指定一个参数 `-parameters`，maven 可以按照下面的配置
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.1</version>
    <configuration>
        <compilerArgument>-parameters</compilerArgument>
        <source>1.8</source>
        <target>1.8</target>
    </configuration>
</plugin>
```

处理的结果:
```
parameter: java.lang.String，name
parameter: int，age
```


### 4. 新的工具

4.1 Jdeps工具：java类的依赖分析工具

4.2 jjs 工具：Nashorm 引擎，可以用来解析javascripts脚本 