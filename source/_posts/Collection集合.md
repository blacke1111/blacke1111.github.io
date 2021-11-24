---
title: Collection接口
date: 2021-11-13 16:53:33
tags: Java集合
categories: java基础
---

## 集合与数组存储数据概述:

集合、数组都是对多个数据进行存储操作的结构，简称ava容器。  
说明:此时的存储，主要指的是内存层面的存储，不涉及到持久化的存储（.txt,.jpg,.avi，数据库中）

## 数组存储的特点:
一旦初始化以后，其长度就确定了。  
数组一旦定义好，其元素的类型也就确定了。我们也就只能操作指定类型的数据了。    
*&emsp;&emsp;比如: Strina[]arr;int[]arr1;obiect[]arr2;


##  数组存储的弊端:
*  —旦初始化以后，其长度就不可修改。
*  数组中提供的方法非常限，对于添加、删除、插入数据等操作，非常不便，同时效率不高。
*  获取数组中实际元素的个数的需求,数组没有现成的属性或方法可用
*  数组存储数据的特点:序、可重复。对于无序、不可重复的需求，不能满足。

# 集合框架

![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211113171108.png)

**Collection接口**：单列集合，用来存储一个一个的对象  
&emsp;&emsp;**List接口** ：存储有序的，可重复的数据  
&emsp;&emsp;&emsp;ArrayList，LinkedList，Vector  
&emsp;&emsp;**Set接口**：存储无序，不可重复的数据  
&emsp;&emsp;&emsp;HashSet，LinkedHashSet，TreeSet  
**Map接口**：双列集合，用来存储一堆（key，value）一对的数据  
&emsp;&emsp;&emsp;HashMap，LinkedHashMap，TreeMap、Hashtable、Properties

## Iterator迭代器接口

* lterator对象称为迭代器(设计模式的一种)，主要用于遍历Collection集合中的元素。
* **GOF给迭代器模式的定义为:提供一种方法访问一个容器(container)对象中各个元素，而又不需暴露该对象的内部细节**。迭代器模式，就是为容器而生。类似于“公交车上的售票员”、“火车上的乘务员”、“空姐”。
* Collection接口继承了java.lang.Iterable接口，该接口有一个iterator()方法，那么所有实现了Collection接口的集合类都有一个iterator()方法，用以返回一个实现了lterator接口的对象。
* **lterator仅用于遍历集合**，lterator木身并不提供承装对象的能力。如果需要创建lterator对象，则必须有一个被迭代的集合。
* **集合对象每次调用iterator()方法都得到一个全新的迭代器对象**，默认游标都在集合的第一个元素之前。

```java  
 @Test
    public void test1(){
        Collection<Object> collection = new ArrayList<>();
        collection.add(13);
        collection.add(false);
        collection.add(new Person("zhang"));
        collection.add(new Person("hao"));
        Iterator<Object> iterator = collection.iterator();
        while (iterator.hasNext()) {
			//next 1。指针下移 2.将下移以后集合位置上的元素返回
            System.out.println(iterator.next());
        }
    }
```
remove方法：

```java  
 @Test
    public void test2() {
        Collection<Object> collection = new ArrayList<>();
        collection.add(13);
        collection.add(false);
        collection.add(new Person("zhang"));
        collection.add(new String("hao"));
        Iterator<Object> iterator = collection.iterator();
        while (iterator.hasNext()) {
            Object next = iterator.next();
            if (next.equals("hao")){
			//移除当前迭代器指针指向元素
                iterator.remove();
            }
        }
        Iterator<Object> iterator1 = collection.iterator();
        while (iterator1.hasNext()) {
            System.out.println(iterator1.next());
        }
    }

```



## List接口
* 鉴于Java中数组用来存储数据的局限性，我们通常使用List替代数组
* List集合类中元素有序、且可重复，集合中的每个元素都有其对应的顺序索引。
* List容器中的元素都对应.一个整数型的序号记载其在容器中的位置，可以根据序号存取容器中的元素。
* JDKAPI中List接口的实现类常用的有:**ArrayList**、**LinkedList** 和**Vector**.  
**List涉及到比较是按照equals方法。**


**面试题:ArrayList、 LinkedList、 Vector三者的异同?**  
同:三个类都是实现了List接口，存储数据的特点相同:存储有序的、可重复的数据

> 不同： 
>>**ArrayList**: 作为List接口的主要实现类，线程不安全的，效率高;底层使用0bject[]eLementData存储  
>>**LinkedList**: 对于频繁的插入、删除操作，使用此类效率比ArrayList高;底层使用双向链表存储  
>>**Vector**:作为List接口的古老实现类；线程安全的，效率低。底层使用0bject[]eLementData存储  


### ArrayList
ArrayList的源码分析:
**jdk 7**情况下  
ArrayList list = new ArrayList();//底层创建了长度是10的object[]数组   eLementDatalist.add(123); //eLementData[size+1] = new Integer(123);  
...  
list.add(11);//如果此次的添加导致底层elementData数组容量不够，则扩容。 
默认情况下，扩容为原来的容量的1.5倍，同时需要将原有数组中的数据复制到新的数组中。|
![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211113193854.png)

**jdk 8**中ArrayList的变化:  
ArrayList list = new ArrayList();//底层object[] elementData初始化为{}.并没有创建数组长度为10的数组  
list.add(123);//第一次调用add()时，底层才创建了长度10的数组，并将数据123添加到eLement数组中。  
...   
后续的添加和扩容操作与jdk 7无异。 


小结: jdk7中的ArrayList的对象创建类似于单例的饿汉式，而jdk8中的ArrayList的对象
创建类似于懒汉式。  


### LinkedList
jdk8：
LinkedList list = new LinkedList();内部声明了Node类型的first和Last属性，默认值为null  
List.add ( 123);//将123封装到Node中，创建了Node对象。
其中，Node定义为:
![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211113200814.png)

## Set接口
* Set接口中没有额外定义新的方法，使用的都是Ccollection中声明过的方法。
* Set接口是Collection的子接口，set接口没有提供额外的方法  
* Set集合不允许包含相同的元素，如果试把两个相同的元素加入同一个Set集合中，则添加操作失败。   
* Set判断两个对象是否相同不是使用==运算符，而是根据equals()方法  
> **Set接口**：存储无序，不可重复的数据  
>>HashSet：作为Set的主要实现类：线程不安全，可以存储null  
>>LinkedHashSet：继承于HashSet,遍历内部结构时，可以按照添加的顺序遍历  
>>TreeSet：添加的数据来自同一个类的实例，**红黑树**，可以按照指定添加对象的属性进行排序。 


### HashSet
1：无序 ： 不等于随机性，存储的数据在底层数组中并非按照数组索引的顺序添加，而是根据数据的哈希值决定的。

2：不可重复性：保证添加的元素按照equals()判断时，不能返回true.即:相同的元素只能添加一个


二、添加元素的过程:以**HashSet**为例:底层实现数组加链表
添加元素a的过程，首先调用a元素的hashSet（）方法，计算出元素a的哈希值，再通过取余找到在底层数组中存放的索引位置，如果位置上没有元素，直接存放上去，如果有就遍历当前数组索引位置上的链表头节点元素，依次比较，比较的方式为：先比较两个元素的hash值是否相同  
不相同:添加元素a相同  
相同：a元素调用equals方法：  
*  返回true添加失败  
* 返回false添加成功  


**jdk7：头插法** &emsp;&emsp; **jdk8：尾插法**  
**总结：七上八下**


**要求1**:向Set中添加的数据，其所在的类一定要重写hashcode( )和equals()  
**要求2**:重写的hashCode()和equals()尽可能保持一致性:相等的对象必须具有相等的散列码  
**重写两个方法的小技巧**:对象中用作equals()方法比较的Field，都应该用来计算hashCode值。

### LinkedHashSet
继承于HashSet  
**为什么输出时可以按照添加的顺序输出：**
![](https://i.loli.net/2021/11/14/gmUEWPaV5v3RuKH.png)

对于频繁的遍历操作，LinkedHashset效率高于HashSet.

### TreeSet（红黑树）
1.向Treeset中添加的数据，要求是相同类的对象

2.两种排序方式:自然排序（实现Comparable接口）和定制排序

3.自然排序中，比较两个对象是否相同的标准为: compareTo()返回0.不再是equals().

```java
//自然排序
//User实现Comparable接口 重写
 @Override
    public int compareTo(Object o) {
        if (o instanceof  User) {
            User o1 = (User) o;
            int i = this.name.compareTo(o1.getName());

            return i != 0 ? i : Integer.compare(this.age, o1.age);
        }
        return 0;
    }

//Test： 比较两个对象是否相同的标准为: compareTo()返回0.不再是equals().
	TreeSet<Object> treeSet = new TreeSet<>();
	treeSet.add(new User("Tom",45));
        treeSet.add(new User("Jack",45));
        treeSet.add(new User("Tom",65));
        treeSet.add(new User("Rose",12));
        treeSet.add(new User("Zhang",2));
        for (Object o : treeSet) {
            User o1=(User)o;
            System.out.println(o1);
        }
```

```java

//定制排序 
//4.定制排序中，比较两个对象是否相同的标准为: compare()返回0.不再是equals().
 @Test 
    public void test2(){

        Comparator<Object> objectComparator = new Comparator<Object>(){
            @Override
            public int compare(Object o1, Object o2) {
                if (o1 instanceof User&& o2 instanceof User){
                    User u1=(User)o1;
                    User u2=(User)o2;
                    return  Integer.compare(u1.getAge(), u2.getAge());
                }else{
                    throw  new RuntimeException("输入的苏剧类型不匹配");
                }
            }
        };
        TreeSet<Object> treeSet = new TreeSet<>(objectComparator);
        treeSet.add(new User("Tom",45));
        treeSet.add(new User("Jack",45));
        treeSet.add(new User("Tom",65));
        treeSet.add(new User("Rose",12));
        treeSet.add(new User("Zhang",2));
        for (Object o : treeSet) {
            User o1=(User)o;
            System.out.println(o1);
        }
    }

```

## Map接口
![](https://i.loli.net/2021/11/14/Mba1OKrq7Zcdes4.png)

**Map**:双列数据，存储key-value对的数据---类似于高中的函数: y = f(x)  
**HashMap**:作为Map的主要实现类;线程不安全的，效率高;存储null的key 和vaLue  
**LinkedHashMap**:保证在遍历map元素时，可以按照添加的顺序实现遍历。  
原因:在原有的HashMap底层结构基础上，添加了一对指针，指向前一个和后一个元素,**对于频繁的遍历效率高于HashMap**  
**TreeMap**:保证按照添加的key-value对进行排序，实现排序遍历。底层实现红黑树  
**HashtobLe**;作为古老的实现类;线程安全的，效率低:不能存储null的key和value  
**Properties**：常用来处理配置文件。key和value都是String类型

**二、Map结构的理解:**  
Map中的key:无序的、不可重复的，使用set存储所有的key  
Map 中的value:无序的、可重复的，使用Collection存储所有的value  
一个键值对: key-value构成了一个Entry对象。  
Map中的entry:无序的、不可重复的，使用set存储所有的entry  


### HashMap（具体细节建议直接看源码）：
jdk7底层结构只有:数组+链表。jdk8中底层结构:数组+链表+红黑树。  
当数组的某一个索引位置上的元素以链表形式存在的数据个数>8且当前数组的长度>64时，此时此索引位置上的所有数据改为使用红黑树存储。|

### LinkedHashMap
主要改变是：
![](https://i.loli.net/2021/11/15/w8ZQP9aBh5nHvs6.png)

可以按照添加的顺序输出。