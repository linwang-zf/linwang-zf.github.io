---
title: SpringMVC系列(二)---RequestMapping中url匹配原理
author: linWang
date: 2022-02-19 18:02:08
tags: SpringMVC系列
categories:	Spring源码
top: true
---
>   本文详解的介绍了@RequestMapping中常见的url匹配类型：精准匹配、模糊匹配。并且分析了这几种类型下url的匹配规则原理

| 属性                       | 说明                             |
| -------------------------- | -------------------------------- |
| MappingRegistry.registry   | 所有的url匹配规则均保存在此Map中 |
| MappingRegistry.pathLookup | 仅保存不含通配符的url匹配规则    |

<!--more-->

在[SpringMVC系列(一)---RequestMapping注解的原理](https://blog.linwang.tech/2022/01/23/RequestMapping%E6%B3%A8%E8%A7%A3%E7%9A%84%E5%8E%9F%E7%90%86)这篇文章中，我们学习了@RequestMapping注解的原理，以及http请求的流程，但是springMVC到底是如何根据request的url请求来匹配到具体的url规则呢，本章我们就来详细的学习一下。
我们知道RequestMapping不仅仅支持精准匹配，还可以进行模糊匹配，即使用* ？ **等通配符，下面我们就分别来看下这两种方式是如何实现的。

## 精准匹配

当我们自定义方法上的@RequestMappingMapping注解中不包含通配符时即为精准匹配，例如下面的匹配规则。

![](示例图.png)

我们一起来看下上述例子中的匹配源码，如下图。

![](1.png)

lookupHandlerMethod函数的作用就是根据给定的lookupPath，查找一个最佳匹配的handleMethod，下面我们继续深入看这个函数。

![](2.png)

上图中红框中的函数getMappingsByDirectPath的作用是获取key为指定url的RequestMappingInfo。当返回结果directPathMatches不为空时，继续调用addMatchingMappings函数，我们继续看该函数的源码。

![](11.png)

addMatchingMappings函数的作用是对比mapping和request，如果匹配结果不为null，则加入matcher列表中。在上述示例的精准匹配中，这里的mappings为

![](3.png)

此时获取到最佳的RequestMappingInfo匹配，并在mappingRegistry中查找对应的MappingRegistration。并返回其中的handleMethod。

## 模糊匹配

### URL中的通配符为：* ? **

有时候我们在开发时，url中往往并不是某个具体的字符串，而是一个模糊的通配符，用来匹配特性相同的一类url。举个例子如下：

![](4.png)

假设我们当前http访问的路径为：“http://localhost:7001/RequestMappingDemoController/vagueMatch”
对于上述的例子，getMappingsByDirectPath函数将无法获取到/RequestMappingDemoController/vagueMatch对应的value值，此时将进入下面的代码中。

![](5.png)

可以看到依旧是调用了addMatchingMappings函数，和精准匹配的区别仅仅在于第一个参数不同，精准匹配中第一个参数为查找到的value，而模糊匹配中传入的是整个mappingRegistry中保存的所有key值，需要遍历去查找最佳的匹配。
匹配的源码如下：

![](6.png)

pathMatcher是一个Spring提供的工具类，用来校验两个字符串是否匹配，支持*，? ,**, {param}等通配符，RequestMapping注解中的通配符匹配规则本质就是通过pathMatcher来实现的。
常见的匹配场景如下：

| pattern    | lookupPath | match |
| ---------- | ---------- | ----- |
| /a/*       | /a/b       | true  |
| /a/?       | /a/b       | true  |
| /a/**      | /a/b       | true  |
| /a/**      | /a/b/c     | true  |
| /a/{param} | /a/b       | true  |
|            |            |       |
| /a/*       | /a         | false |
| /a/*       | /a/b/c     | false |
| /a/?       | /a         | false |
| /a/?       | /a/b/c     | false |
| /a/{param} | /a         | false |

### URL中的通配符包含：{param}

通过{param}的方式，我们可以获取url路径中的参数，那么SpringMVC又是怎么做的呢？

![](7.png)

通过addMatchingMappings函数获取到最佳的url匹配规则后，会通过handleMatch对匹配结果Match进行处理，我们继续深入看下HandleMatch源码。

![](10.png)	

在handleMatch函数中，调用了extractMatchDetails函数，通过函数名我们可以了解该函数是用来提取Match详情的，我们通过源码来看下其真正的作用：

![](8.png)

在extractMatchDetails函数中，调用了PathMatcher工具类中的extractUriTemplateVariables函数，这个函数的入参有两个，一个是url匹配规则，一个是request中的url访问路径，extractUriTemplateVariables可以从url匹配规则中提取参数名作为key，从url访问路径中提取参数值作为value，存放在Map中，并将Map暴露在Request的Attribute中，如下图。

![](9.png)

## 总结

在上述的篇幅中，我们重点介绍了SpringMVC针对不同类型的url匹配规则是如何获取HandlerMethod的，以及在过程中具体做了哪些操作等。

- 精确匹配
    - 从MappingRegistry.pathLookup中直接获取指定的handlerMethod
- 模糊匹配
    - 从MappingRegistry.registry中获取模糊的url匹配规则(根据PathMatch工具类来实现)
    - 如果url匹配规则中包含{param}，则通过handleMatch获取param对应的参数中，并将其暴露在Request的Attribute中

在获取到HandlerMethod后，下一步就是要执行真正的函数了，但是在执行之前，有一步最重要的过程，就是参数解析和映射，这一块内容的详细讲解请看[SpringMVC系列(三)---参数解析流程](https://blog.linwang.tech/2022/02/19/SpringMVC%E7%B3%BB%E5%88%97(%E4%B8%89)---%E5%8F%82%E6%95%B0%E8%A7%A3%E6%9E%90%E6%B5%81%E7%A8%8B/#more)文章。
