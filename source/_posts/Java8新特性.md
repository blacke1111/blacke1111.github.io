---
title: Java8新特性
date: 2022-06-26 20:54:35
tags: java
---



# Lambda表达式的使用:



```java
public class LambdaTest {


    //语法格式一: 无参，无返回值
    @Test
    void test1(){
        Runnable r2=()->{
            System.out.println("hello");
        };
        r2.run();
    }
    
    //语法格式二：Lambda需要一个参数，但是没有返回值
    
    @Test
    void test2(){
        Consumer<String> consumer=(String s1)->{
            System.out.println(s1);
        };
    }
    //语法格式三： 参数类型可以省略，因为可由编译器推断得出，称为"类型推断"
    void test3(){
        //1
        Consumer<String> consumer=(s1)->{
            System.out.println(s1);
        };
        //2
        ArrayList<Object> objects = new ArrayList<>();
        //3
        String arr[]={"2","3","1"};
        
    }
    //语法格式四：Lambda若只需要一个参数时，参数的小括号可以省略
    @Test
    void test4(){
        Consumer<String> consumer=s1-> System.out.println(s1);;
    }
    //语法格式五：需要两个或以上的参数，多执行语句，并且可以有返回值
    @Test
    void test5(){
        Comparator<Integer> com2=(o1, o2) ->{
            System.out.println(o1);
            System.out.println(o2);
            return o1-o2;
        };
    }
    //语法格式六：当lambda体只有一条语句时，return与大括号都可以省略
    @Test
    void test6(){
        Comparator<Integer> com2=(o1, o2) -> o1-o2;
    }

}
```

# 函数式接口：

![image-20220626213039844](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220626213039844.png)

![image-20220626213350091](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220626213350091.png)

# 方法引用：

```java
/**
 * 方法引用的使用
 * 1.使用的情景: 
 *  
 *  对象：：非静态方法
 *  类：：静态方法
 *  
 *  类：：非静态方法
 *  
 *  方法引用使用要求：要求接口中的抽象方法的形参列表和返回值类型与方法引用的方法的形参列表和返回值类型相同！
 */
public class MethodRefTest {
    
    //情况一： 对象::实例方法
    //Consumer中 void accept(T t)
    //printStream 中的void println(T t)
    @Test
    void test1(){
        Consumer<String> con1=str-> System.out.println(str);
        con1.accept("aaa");

        System.out.println("***********");
        PrintStream ps = System.out;
        Consumer<String> con2=ps::println;
        con2.accept("asd");
        
    }
    
    //Supplier中的T get()
    //Employee中的String getName()
    @Test
    void test2(){
        Employee emp = new Employee(1001, "Tom", 23, 5600);
        Supplier<String> sup1=emp::getName;
        EmployeeData employeeData = new EmployeeData();
        employeeData.getEmployee();
    }
    
    
    //情况二： 类：：静态方法
    //comparator中的int compare(T t1,T t2)
    //Integer中的int compare(T t1,T t2)
    @Test
    public void test3(){
        Comparator<Integer> com1=Integer::compareTo;
    }
    
    //情况三: 类：：实例方法(有难度)
    //consumer中的int compare(T t1,T t2)
    //String中的int t1.compareTo(t2)
    @Test
    void test4(){
        Comparator<String> con2=String::compareTo;
    }
    
    //Function中的R apply(T t)
    //Employee中的String getName()
    @Test
    void test5(){
        Function<Employee,String> function= Employee::getName;
    }
    
    //构造器引用
    @Test
    void test6(){
        Supplier<Employee> supplier=Employee::new;
    }
    
    //数组引用
    //Function中的R apply(T t)
    @Test
    void test7(){
        Function<Integer,String[]> function=length ->new String[length];

        Function<Integer,String[]> function2=String[] ::new;
    }
    
}
```

# 强大的StreamAPI



集合讲的是数据，Stream讲的是计算

1- 创建 Stream

一个数据源

2- 中间操作

一个中间操作链,对数据源的数据进行处理

3-终止操作(终端操作)

一旦执行终止操作,就执行中间操作链,并产生结果。之后，不会再被使用

创建Stream流:

```java
//创建 Stream 方式一: 通过集合
@Test
public void test1(){
    List<Employee> employees = EmployeeData.getEmployee();
    //default Stream<E> stream()返回一个顺序流
    Stream<Employee> stream = employees.stream();
    //default Stream<E> parallelStream(): 返回一个并行流
    Stream<Employee> parallelStream = employees.parallelStream();
}

//创造方式二
@Test
public void test2(){
    int []arr={1,2,3,4,5,6,7};
    Arrays.stream(arr);
}
//创造方式三
@Test
public void test3(){
    Stream<Integer> stream = Stream.of(1, 2, 3, 4, 5, 6);
}

//创建方式四：创建无限流
@Test
public void test4(){
    //遍历前十个偶数
    Stream.iterate(0,t->t+2).limit(10).forEach(System.out::println);

    //生成
    Stream.generate(Math::random).limit(10).forEach(System.out::println);

}
```

筛选与切片

![image-20220626225007702](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220626225007702.png)

```java
@Test
    public void test1(){
        List<Employee> list = EmployeeData.getEmployee();

        //filter(Predicate p)- 接受lambda，从流中排除某些元素
        list.stream().filter(e->e.getSalary()>7000).forEach(System.out::println);
        System.out.println();
        //limit(n)-截断流，使其元素不超过给定数量
        list.stream().limit(3).forEach(System.out::println);
        System.out.println();
        //skip(n)- 跳过元素，返回一个扔掉了前n个元素流。若流中元素不足n个，则返回
        list.stream().skip(3).forEach(System.out::println);
        System.out.println();

        //distinct()-筛选，通过流所生成元素的hashCode()和equals()去重重复元素
        list.stream().distinct().forEach(System.out::println);
    }
```



映射：

![image-20220626230631647](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220626230631647.png)

```java
@Test
    public void test2(){
        //map
        List<String> list = Arrays.asList("aa", "bb", "cc");
        list.stream().map(String::toUpperCase).forEach(System.out::println);

        System.out.println();

        //map
        List<Employee> employees = EmployeeData.getEmployee();
        employees.stream().map(Employee::getName).filter(name->name.length()>3).forEach(System.out::println);
        System.out.println();
        //flatMap
        list.stream().map(StreamAPITest1::fromStringToStream).forEach(System.out::print);
        //输出结果为:java.util.stream.ReferencePipeline$Head@200a570f java.util.stream.ReferencePipeline$Head@16b3fc9e java.util.stream.ReferencePipeline$Head@e2d56bf
                    
        System.out.println("**");
        list.stream().flatMap(StreamAPITest1::fromStringToStream).forEach(System.out::print);
        //输出结果为：a a b b c c

        //mapToDouble
        List<String> list1 = Arrays.asList("231", "23", "434.5");
        double[] doubles = list1.stream().mapToDouble(Double::parseDouble).toArray();
        for (double aDouble : doubles) {
            System.out.print(aDouble+",");
        }

    }
```

排序:

![image-20220626233447839](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220626233447839.png)

匹配和查找

![image-20220627103935831](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220627103935831.png)

```java
 @Test
public void test1(){
    //allMatch
    List<Employee> employees = EmployeeData.getEmployee();
    boolean b = employees.stream().allMatch(e->e.getAge()>18);
    System.out.println(b);
    //anyMatch
    boolean b1 = employees.stream().anyMatch(e -> e.getAge() > 18);
    System.out.println(b1);
    //noneMatch
    boolean b2 = employees.stream().noneMatch(e -> e.getName().startsWith("雷"));
    System.out.println(b2);
    //findFirst
    Optional<Employee> employee = employees.stream().sorted((e1, e2) -> e1.getAge() - e2.getAge()).findFirst();
    System.out.println(employee);
    //count
}
```



归约

![image-20220627105217837](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220627105217837.png)

**T iden 表示初始值**

```java
//归约
@Test
public void test3(){
    List<Integer> list = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
    Integer reduce = list.stream().reduce(0, Integer::sum);
    System.out.println("reduce = " + reduce);

}
```



![image-20220627105805032](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220627105805032.png)

```java
```



## Collectors:

![image-20220627110118590](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220627110118590.png)

## Optional:

![image-20220627112012699](https://edu-1395430748.oss-cn-beijing.aliyuncs.com/images/imgs/image-20220627112012699.png)
