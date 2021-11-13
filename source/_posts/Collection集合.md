---
title: Collection接口
date: 2021-11-13 16:53:33
tags: Java集合
categories: java基础
---

## 1．集合与数组存储数据概述:

集合、数组都是对多个数据进行存储操作的结构，简称ava容器。  
说明:此时的存储，主要指的是内存层面的存储，不涉及到持久化的存储（.txt,.jpg,.avi，数据库中）

## 2．数组存储的特点:
一旦初始化以后，其长度就确定了。  
数组一旦定义好，其元素的类型也就确定了。我们也就只能操作指定类型的数据了。    
*&emsp;&emsp;比如: Strina[]arr;int[]arr1;obiect[]arr2;


## 3．数组存储的弊端:
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



**面试题:ArrayList、 LinkedList、 Vector三者的异同?**  
同:三个类都是实现了List接口，存储数据的特点相同:存储有序的、可重复的数据

> 不同： 
>>**ArrayList**: 作为List接口的主要实现类，线程不安全的，效率高;底层使用0bject[]eLementData存储  
>>**LinkedList**: 对于频繁的插入、删除操作，使用此类效率比ArrayList高;底层使用双向链表存储  
>>**Vector**:作为List接口的古老实现类；线程安全的，效率低。底层使用0bject[]eLementData存储  


### 1.ArrayList
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


### 1.LinkedList
jdk8：
LinkedList list = new LinkedList();内部声明了Node类型的first和Last属性，默认值为null  
List.add ( 123);//将123封装到Node中，创建了Node对象。
其中，Node定义为:
![](https://gitee.com/haoyumaster/imageBed/raw/master/imgs/20211113200814.png)
