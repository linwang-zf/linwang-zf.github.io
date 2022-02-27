---
title: Java注解
author: linWang
date: 2022-02-27 16:32:28
tags: Java系列----注解
categories:	Java系列
---

>   本文介绍了注解的基本用法，包括注解的定义、属性以及常用的 元注解和Java内置的注解。

<!--more-->

## 注解的定义

注解是在Java SE5.0版本后开始引入的概念，同Class和Interface一样，注解也是一种类型。

```java
public @interface customAnnotation{
}
```

从注解的定义上可以看出，其实注解的写法和接口很像，只是前面多了一个@符号而已。

## 元注解

在介绍注解之前，我们先来介绍下什么是元注解。简单的来说，元注解就是可以应用在其他注解上的注解，是一种基本注解。举个例子：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {
}
```

@Override这个注解想必大家都不陌生，是一种标记注解，用来表明当前方法是子类重写父类的方法。

这个注解上面有两个注解：@Target、@Retention，这两个注解就是元注解。除了这两种外，常见的元注解还有：@Inherited、@Documented、@Repeatable。下面对这些注解进行一下简要的介绍。

### @Retention

当@Retention注解到其他注解上时，定义了这个注解的存活时间。他的取值如下所示：

*   RetentionPolicy.SOURCE：表明注解只在源码阶段有效，编译器在编译时将忽视该注解，即不会保留在编译好的class文件中。
*   RetentionPolicy.CLASS(**默认**)：表明注解不会被加载到JVM内存中，仅会在源码和Class文件中有效。
*   RetentionPolicy.RUNTIME：在运行期时该注解的信息也存在，即可以通过Java的反射机制获取到该注解的基本信息。

### @Target

@Target可以用来约束注解可以应用的地方，常用的取值如下所示：

*   ElementType.FIELD ：表明该注解可以作用在字段域。
*   ElementType.TYPE：表明该注解可以作用在类、接口上。
*   ElementType.METHOD：表明该注解可以作用在方法上。
*   ElementType.PARAMETER：表明该注解可以作用在参数声明上。
*   ElementType.CONSTRUCTOR：表明该注解可以作用在构造方法上
*   ElementType.ANNOTATION_TYPE：表明该注解可以作用在另一个注解上。
*   ElementType.PACKAGE ：表明该注解可以作用在包上。
*   ElementType.LOCAL_VARIABLE：表明该注解可以作用局部变量上。

### @Documented

@Documented指定了注解的基本信息会出现在JavaDoc的文档中。

### @Inherited

Inherited的意思是继承，但是并不表示注解可以继承，而是表示如果注解A被@Inherited修饰，那么注解A作用的类，其子类也是继承该注解A。

### @Repeatable

Repeatable的意思是可重复，是Java8之后引入的新特性。可以在同一个类上重复应用相同的某个注解。举个例子：

*   Animal注解：接受一个动物的名字name

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Repeatable(Animals.class)
public @interface Animal {
    String name() default "";
}
```

*   注解容器：用来保存多个Animal的容器

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface Animals {
    Animal[] value();
}
```

*   测试类

```java
@Animal(name = "cat")
@Animal(name = "dog")
@Animal(name = "bird")
public class TestDemo {

    public static void main(String[] args) {
        Class<TestDemo> clazz = TestDemo.class;
        //1. 通过getDeclaredAnnotationsByType方法也可以获取
        //Animal[] annotationsByType = clazz.getDeclaredAnnotationsByType(Animal.class);
        
        //2. 通过getAnnotation无法获取到Animal注解的信息
        //Animal annotation = clazz.getAnnotation(Animal.class);
        Animal[] annotationsByType = clazz.getAnnotationsByType(Animal.class);
        System.out.println(annotationsByType.length);
        for (Animal animal : annotationsByType) {
            System.out.println(animal.name());
        }
    }
}

//output
3
cat
dog
bird
```

## 注解属性

注解中只有属性，没有方法，但是属性的定义是以“无参数的方法来声明的”，方法的名字就是属性的名字，方法的返回值就是属性的类型。

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Repeatable(Animals.class)
public @interface Animal {
    String name() default "";
}
```

上述自定义的@Animal注解中定义了一个属性name，类型是String，默认值为空字符串。使用方式如下：

```java
@Animal(name = "cat")
public class TestDemo {
}
```

只需要在需要的类上增加其中@Animal即可，并可以自定义name的值，如果@Animal的属性值为value，则可以省略不写。4

## 注解的原理

待补充（代理实现）

## Java内置的注解

### @Override

表示@Override修饰的方法是子类重写父类的

### @Deprecated

用来修饰某个类、方法已经过时，可能在之后的JDK版本后会删除。

### @SuppressWarnings

可以用来消除编译器产生的警告信息
