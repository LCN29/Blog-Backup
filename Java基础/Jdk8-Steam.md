# [Java基础] Jdk8 - Steam

## 1. Stream 是什么?

Stream 是对集合（Collection）对象功能的增强。 它专注于对集合对象各种高效的聚合操作（aggregate operation）[注: 和数据库中的聚合函数类似，如求平均值，最大值，前几个等操作]，或者大批量数据操作 (bulk data operation)。


## 2. Stream 的特点
1. Stream不存储数据  
Stream 不是集合元素，它不是数据结构并不保存数据，它是有关算法和计算。 如果硬要说的话，它更像一个高级版本的 Iterator。 我们平时使用的 Iterator，只能显式地一个一个遍历元素并对元素进行我们想要的操作。 但是 Stream，只要给出需要对其包含的元素执行什么操作，比如 "过滤掉长度大于 10 的字符串"，Stream 会隐式地在内部进行遍历，做出相应的数据转换。

2. Stream不改变源数据
当我们将一个集合转为 Stream，然后对 Stream 进行的任何操作，我们的数据源不会受到任何影响。

3. Stream 具有延迟执行的特性。 Stream 流的很多操作如 filter，map 等中间操作(intermediate operations，返回值还是一个 Stream，因此可以通过链式调用将中间操作串联起来)是延迟执行的，只有到最终操作(terminal operation，只能返回void或者一个非stream的结果)才会将操作顺序执行。
```java 
public void filter(Student stu) {
	System.out.println("开始过滤");
	return stu.getScore() > 80;
}

public void test() {
    
    List<Student> stuList = new ArrayList<>();

    stuList.add(new Student(21));
    stuList.add(new Student(21));
    stuList.add(new Student(21));

    // 此处的 filter 就是我们说的 中间操作
	Stream<Student> stream = stuList.stream().filter(this::filter);
	System.out.println("准备开始了");
	// collect 就是我们说的终点操作
	List<Student> stus =  stream.collect(Collectors.toList());
}


// 通过日志可知： filter方法是在 collect 执行时才会执行

/** 
 *	输出日志：   
 *  准备开始了
 *  开始过滤
 *	开始过滤	
 * 	开始过滤
 */
```

4. 对于 Stream 的聚合、消费或收集操作只能进行一次，再次操作会报错
```java

    // generate 不断生成字符串，Limit(20) 限制 20 条。
	Stream<String> stream = Stream.generate(()->"user").limit(20);
	// 正常执行
	stream.forEach(System.out::println);

	// 报错：同一个stream对象只能进行一次
	stream.forEach(System.out::println);
```

5. Steam 的数据源本身可以是无限的。 除了上面说的中间操作和最终操作，其中还有一种短路操作(short-circuiting)。
* 对于一个中间操作，如果它接受的是一个无限大的 Stream，但返回一个有限的新 Stream，比如 limit(限制前多少个)
* 对于一个最终操作，如果它接受的是一个无限大的 Stream，但能在有限的时间计算出结果，比如findFirst(找第一个)


## 3. Stream 的使用

#### 1. Stream 的创建

(1) List 转为Stream
```java

	List<Student> stuList = new ArrayList<>();

	// 普通的流
	Stream<Student> stream = stuList.stream();

	// 并行流
	Stream<String> stream1 = strs.parallelStream();
```

(2) 2.通过数组创建
```java

	// 基本类型
	int[] arr = new int[]{1，2，34，5};
	IntStream intStream = Arrays.stream(arr);

	// 引用类型
	Student[] studentArr = new Student[]{new Student("s1"，29)，new Student("s2"，27)};
	Stream<Student> studentStream = Arrays.stream(studentArr);
```

(3) 数据集合
```java

	Stream<Student> stream = Stream.of(new Student("小红"，20)，new Student("小明"，10));

	Student[] studentArr = new Student[]{new Student("s1"，29)，new Student("s2"，27)};
	Student[] studentArr2 = new Student[]{new Student("s1"，29)，new Student("s2"，27)};

	// Stream.of  里面的数据类型是什么，那么返回的Stream的泛型就是什么
	Stream<Student[]> stream01 = Stream.of(studentArr，studentArr2);

```

(4) 创建空的流
```java
	
	// 创建 空的流
	Stream<Student> s = Stream.empty();

	Stream<Integer> s = Stream.empty();
```

(5) 创建无限流
```java
	
	// 通过limit 截取需要的个数
	Stream.generate(()->new Student("name"，10)).limit(20).forEach(System.out::println);

```

(6) 创建规律的无限流
```java

	// 0: 基点，从0开始，生成 0，1，2，3 ...顺序的10个
	Stream.iterate(0，x->x+1).limit(10).forEach(System.out::println);

	// 0：基点，从0开始，生成 0，0，0  10个数
	Stream.iterate(0，x->x).limit(10).forEach(System.out::println);

```

#### 2.Stream转为集合

(1) toList(): 转为List
```java
	List<Student> list = stuList.stream().collect(Collectors.toList());
```

(2) toMap(): 转为Map
```java
	// 参数1： key  参数2： value  参数三: 当map的key相同时：如何处理  s--> 新的值(s遍历的值)  a--> 已有的和新的值key相同的对象(已有的值) 
	Map<Integer，String> map = Arrays.stream(students).collect(toMap(Student::getScore，Student::getName，(s，a)->{
			return s + a ;
	}));
```

(3) toSet(): 转为Set
```java
	Set<Student> set = Arrays.stream(students).collect(toSet());
```

(4) toArray(): 转为Array
```java
	Student[] s = Arrays.stream(students).toArray(Student[]::new);
```

(5) toCollection(): 转为指定的Collection对象
```java
	HashSet<Student> s = Arrays.stream(students).collect(toCollection(HashSet::new));
```


#### 3.对流的操作

* 1.筛选  

(1) filter(): 条件过滤
```java
	// filter里面的lambda表达式 返回一个Boolean值，如果为true 将这个对象保留
	stuList.stream().filter(x -> x.getScore() > 80).collect(Collectors.toList());
```

(2) distinct()：去重
```java

	// stuList中重复的项会被去掉
	stuList.stream().distinct().collect(Collectors.toList());

```


* 2.截取  

(1) limit(): 截取前几个
```java
	// 获取stuList中的前3个
	stuList.stream().limit(3L).collect(Collectors.toList());
```

(2) skip(): 跳过前几个
```java
	
	// 去掉stuList中的前3个
	stuList.stream().skip(3L).collect(Collectors.toList());
```

* 3.转换(映射)  

(1) map(): 对象转换
```java
	
	List<Student> stus = stuList.stream().collect(Collectors.toList());

	// 通过map 将 输出结果转为 List<String>  值为每个对象的 name属性
	List<String> strList = stuList.stream().map(item -> item.getName()).collect(Collectors.toList);
```

(2) flatMap(): 将每个值转换成另一个流，然后将所有的流连起来
```java
	
	String[] arr1 = {"a"，"b"，"c"，"d"};
    String[] arr2 = {"e"，"f"，"c"，"d"};

     // 返回结果 List<String[]> 
    Stream.of(arr1，arr2).collect(Collectors.toList);

     //  返回结果 List<String>  2个数组合并成1个
    Stream.of(arr1，arr2).flatMap(Arrays::stream).collect(Collectors.toList());
```

* 4.查找  

(1) findFirst(): 查找第一个
```java

	// orElse(): 如果数据没有返回 nothing
	String str = Stream.of(arr).parallel().filter(x -> x.length() > 3).findFirst().orElse("nothing");
```

(2) findAny(): 查找一个(findFirst() 返回的是第一个，findAny() 不一定返回第一个)
```java
	Optional<String> optional = Stream.of(arr).parallel().filter(x -> x.length() > 3).findAny();
	optional.ifPresent(System.out::println);
```

* 5.匹配 

(1) anyMatch(): 是否包含匹配元素
```java
	Boolean aBoolean = Stream.of(arr).anyMatch(x->x.startsWith("a"));
```

(2)allMatch()：是否全部符合
```java
	// 有一个不符合就false
	Boolean aBoolean = Stream.of(arr).allMatch(x->x.startsWith("a"));
```

(3)noneMatch(): 是否存在不符合条件
```java
	// 是否存在 不大于10的
	Boolean result = Stream.of(1，2，3，4，5).noneMatch( x -> x > 10);
```

* 6.遍历 

(1) forEach(): 对流的数据进行遍历操作
```java
	stuList.stream().forEach(item -> System.out.println(item.getName()));
```

* 7.流数据操作 

(1) peek(): 对流中的每个数据进行需要的操作
```java
	//对数据里面的每一项减去 100
	stuList.stream().peek(item -> item.setScore(item.getScore() - 100)).collect(toList());
```

* 8.流合并  

(1) concat(): 2个相同类型的流进行合并
```java

	Stream<String> stream1 = Stream.of(arr1);
    Stream<String> stream2 = Stream.of(arr2);

    Stream.concat(stream1，stream2).distinct().forEach(System.out::println);
```

* 9.规约(将流中的数据，按照需要，归并成一个值)  

(1) reduce(): 归并操作的函数
```java
	// 求和   参数1 提供给归并操作的起始值，参数2: 数据的处理方式，结果返回一个值
	int sum1 = numbers.stream().reduce(0，(a，b) -> a + b);

	int sum2 = numbers.stream().reduce(0，Integer::sum);
```

* 10.数学操作    

(1) sorted(): 排序
```java

	// 按照comparing里面的值进行顺序排序
	stuList.stream().sorted(Comparator.comparing(Student::getScore)).collect(Collectors.toList);

	// 逆序排序
	stuList.stream().sorted(Comparator.comparing(Student::getScore).reversed()).collect(Collectors.toList);

```

(2) max()/min(): 最大值/最小值
```java
	// max 返回了Optional<T>
	Stream.of(arr).max(Comparator.comparing(String::length)).ifPresent(System.out::println);

	// min 返回了Optional<T>
	Stream.of(arr).min(Comparator.comparing(String::length)).ifPresent(System.out::println);
```

(3) count(): 统计
```java
	int count =	Stream.of(arr).count();
```

* 11.分组  

(1) groupingBy(): 按照条件进行分组
```java
	
	// 按照 相同的score一组 进行分组
	Map<Integer，List<Student>> groups = list.stream().collect(Collectors.groupingBy(Student::getScore));


	// 2级分组
	Map<Integer，Map<String，List<Student>>> twoGroup = list.stream()
				.collect(Collectors.groupingBy(Student::getScore，Collectors.groupingBy(Student::getName)));

	// 可以继续多级分组


	// 第二个参数也可以传递其他参数 Long : 个数
	Map<Integer，Long> groups = students.stream().collect(Collectors.groupingBy(Student::getScore，Collectors.counting()));	

	// Long 这一组的总分数
	Map<Integer，Integer> groups = students.stream().collect(Collectors.groupingBy(Student::getScore，summingInt(Student::getScore)));	

	// 是否有最大值
	Map<String，Optional<Student>> = students.stream().collect(Collectors.groupingBy(Student::getScore，maxBy(Comparator.comparing(Student::getScore))));	

	// 转为Set
	Map<String，Set<Integer>> = students.stream().collect(Collectors.groupingBy(Student::getScore，mapping(Student::getScore，toSet())));	
```

* 12.分区(分组的特殊情况，k值为true/false) 

(1) partitioningBy(): 按照条件进行分组
```java
	Map<Boolean，List<Student> groups = list.stream().collect(Collectors.partitioningBy(student -> student.getScore()> 90));
```


* 13.统计(IntSummaryStatistics)  

(1) 使用IntSummaryStatistics进行数学操作
```java
	IntSummaryStatistics summaryStatistics = Arrays.stream(students).collect(Collectors.summarizingInt(Student::getScore));

	System.out.println("getAverage->"+summaryStatistics.getAverage());
    System.out.println("getMax->"+summaryStatistics.getMax());
    System.out.println("getMin->"+summaryStatistics.getMin());
    System.out.println("getCount->"+summaryStatistics.getCount());
    System.out.println("getSum->"+summaryStatistics.getSum());
```

#### 4.并行流
只需要调用顺序流的 parallel() 方法，就可以将普通顺序执行的流转变为并行流
1. 顺序流： 所有的顺序按照顺序进入流，在这个数据被流过滤掉，或者输出，后下个数据才能进入流，进行处理
2. 并行流： 开启多个线程(线程数由电脑决定)，并行的处理这些数据

3. 并行流对 sorted()，distinct() 类似的元素操作api没有影响
4. 并行流对 sorted()，distinct() 类似的元素操作api会比顺序流耗时
5. 并行流对数据的数据的输出是无序的，如果需要有序的输出请用：forEachOrdered
```java
	
	// 顺序流： 0，1，2 ...
	Stream.iterate(0，x -> x + 1 ).limit(10).forEach(System.out::println);

	// 并行流： 3，1，0 ...
	Stream.iterate(0，x -> x + 1 ).limit(10).parallel().forEach(System.out::println);

	// 并行流： 0，1，2
	Stream.iterate(0，x -> x + 1 ).limit(10).parallel().forEachOrdered(System.out::println);
```

#### 5.原始类型流
在数据量比较大的情况下，将基本数据类型（int，double...）包装成相应对象流的做法是低效的，因此，我们也可以直接将数据初始化为原始类型流，在原始类型流上的操作与对象流类似

1. 原始流的生成
```java

	DoubleStream doubleStream = DoubleStream.of(0.1，0.3，0.4);
	IntStream intStream = IntStream.of(1，3，4，5);

	// 生成 [0，100]
	IntStream intStream2 = IntStream.rangeClosed(0，100);

	// 生成 [0，100)
	IntStream intStream3 = IntStream.range(0，100);

```

2. 原始流和对象流的转换
```java
	Stream<Double> stream = doubleStream.boxed();

	DoubleStream doubleStream = stream.mapToDouble(Double::new);
```

#### 6. Optional
通常聚合操作会返回一个 Optional 类型，Optional 表示一个安全(安全指的是避免直接调用返回类型的 null 值而造成空指针异常)的指定结果类型，调用 optional.isPresent() 可以判断返回值是否为空，或者直接调用 ifPresent(Consumer<? super T> consumer) 在结果部位空时进行消费操作；调用 optional.get() 获取返回值
```java
	
	// 创建了一个空的Optional
	Optional<String> opt1 = Optional.empty();
	// false
	System.out.println(opt1.isPresent());

	// 类似给予了一个初始值
	Optional<String> opt = Optional.of("andy with u");
	// true
	System.out.println(opt.isPresent());

	// 获取Optional的默认值
	String result = opt.get();

	// 如果Optional有值进行操作
	opt.ifPresent(item -> System.out.println(item));
```

1. 给 Optional 指定没有值时
```java

	int a = Stream.of(1，3，4).filter(item -> item > 10).max(Comparator.naturalOrder()).orElse(100);

	int a = Stream.of(1，3，4).filter(item -> item > 10).max(Comparator.naturalOrder()).orElseGet(()->-1);

	int a = Stream.of(1，3，4).filter(item -> item > 10).max(Comparator.naturalOrder()).orElseThrow(RuntimeException::new);

```

2. Optional 的创建
```java
	Optional<Student> studentOptional = Optional.of(new Student("user1"，21));

	// 转换 返回值--->Option<返回值>
	Optional<String> optionalStr = studentOptional.map(Student::getName);

	// 迭代(后者覆盖前者) 方法fn 返回 Option<String>，fn2返回Option<String> 要求返回值为 Option<T>，替代前者
	Optional<String> opt = fn().flatMap(fn2());
```


## 4. 参考
[Java 8 中的 Streams API 详解](https://www.ibm.com/developerworks/cn/java/j-lo-java8streamapi/index.html)