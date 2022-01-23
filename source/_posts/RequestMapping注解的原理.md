---
title: RequestMapping注解的原理
author: linWang
date: 2022-01-23 17:32:08
tags: spring注解
categories: Spring系列
---

>   本文将从源码的角度分析@RequestMapping注解的作用，以及不同的url请求是如何路由到对应的method中的

|             类名             |                           简要描述                           |
| :--------------------------: | :----------------------------------------------------------: |
| RequestMappingHandlerMapping |   用来解析@RequestMapping或者@Controlleriu修饰的类或者方法   |
| AbstractHandlerMethodMapping |             在解析过程中大多数的方法都是在此类中             |
|      RequestMappingInfo      |         保存方法或者类上@RequestMapping注解中的信息          |
|        HandlerMethod         |     @RequestMapping注解修饰的方法最终被解析为的目标对象      |
|       MappingRegistry        | 解析后url到RequestMappingInfo、RequestMappingInfo到HandlerMethod的映射均保存在该类中 |

<!--more-->

​	在SpringBoot中，我们只需要在Controller的类和方法上增加一个@RequestMapping注解，就很容易的提供一个url匹配规则，这究竟是怎么做到的？下面我们从两个方面来分析。

*   应用启动时被@RequestMapping注解的类或者方法会被怎么处理？
*   SpringBoot又是如何将http请求正确的路由到指定的方法上？

## @RequestMapping --- 应用启动

​	这个时候，就必须要提到SpringMVC中一个很重要的类了 -- RequestMappingHandlerMapping类，这个类的作用是将我们的Controller中的方法转换为一个RequestMappingInfo类，首先我们先来看一下RequestMappingHandlerMapping的类图：

![RequestMappingHandlerMapping的类图](image-20220123180434628.png)

​	从类图中可以看到，RequestMappingHandlerMapping类实现了InitializingBean接口，这个接口是Spring管理Bean的一个生命周期，实现了这个接口的Bean，spring会在实例化后执行该Bean的afterPropertiesSet()方法。再看这个方法的源码之前，我们要先思考一个问题，RequestMappingHandlerMapping是如何交给Spring来管理的呢？

### RMHM的注入

​	通过全局搜索，发现RequestMappingHandlerMapping会在类WebMvcAutoConfiguration中通过@Bean的方式注入，那WebMvcAutoConfiguration又是怎么注入的呢，其实通过这个类的名字就可以看出来了，这个类是通过SpringBoot提供的AutoConfiguration的机制注入的，这个机制今天不展开来讲，简单的说，就是SpringBoot会在启动的时候，扫描全局的spring.factories文件，并将其中的实现类注入spring容器中，如图所示：

![WebMvcAutoConfiguration注入原理](image-20220123183240651.png)

### RMHM的初始化

​	回到我们之前提到的afterPropertiesSet方法，我们一起来看看这个方法中做了哪些事情？

![afterPropertiesSet源码](image-20220123184544242.png)

​	方法中主要是对RequestMappingInfo中配置参数的一些初始化，重点我们关注红框中的内容，可以看到其调用了super的afterPropertiesSet方法，我们继续深入查看其源码。

![super的afterPropertiesSet源码](image-20220123185209742.png)

​	从上图中可以看出，super类中的afterPropertiesSet调用了initHandlerMethods，这个方法主要干了两件事：

（1）获取spring容器中的所有bean

（2）遍历这些bean，判断是否为isHandler()，如果为true，则执行detectHandlerMethods方法。

isHandler的判断逻辑为(类上是否有Controller或者RequestMapping注解)：

![isHandler方法](image-20220123190959468.png)

​	当检测到类上有Controller或者RequestMapping注解时，进入下面的方法，重点关注红框中的selectMethods方法。

![detectHandlerMethods方法](image-20220123191213205.png) 

​	MethodIntrospector.selectMethods方法的含义是：遍历指定class类中的方法，对每个方法执行inspect函数，并返回method -> inspect返回值的Map集合。对于上图中detectHandlerMethods中的selectMethods，inspect的返回值为RequestMappingInfo类。

​	下面我们一起来看下getMappingForMethod中的具体内容：

![getMappingForMethod的源码](image-20220123193153797.png)

​	getMappingForMethod函数分为三个步骤：

（1）将method上的RequestMapping注解封装为RequestMappingInfo

（2）将method所在类上的RequestMapping注解封装为RequestMappingInfo

（3）将两个结合在一起(结合在一起是为了将url的匹配规则结合起来)

我们继续看detectHandlerMethods后面的代码，可以看到，后面就是将每个method注册为HandlerMethod了。

![注册HandlerMethod](image-20220123193720661.png)

从上图中的代码中，我们就可以发现，handlerMethod被保存在了mappingRegistry中，继续查看register方法。

![registry函数](image-20220123193843918.png)

​	registry方法的主要步骤如下：

（1） 根据类和method创建一个handlerMethod类

（2）将RequestMappingInfo -> HandlerMethod的映射放入mappingLookup中

（3） 将url -> RequestMappingInfo 的映射放入urlLookup中

至此，应用启动过程中@RequestMapping注解的类的解析过程就完成了，我们总结一下主要的步骤

（1）在RequestMappingHandlerMapping启动的过程执行afterPropertiesSet方法

（2）遍历容器中的所有bean，如果该bean存在@Controller或者@RequestMapping注解，则解析该bean

（3）解析该bean中所有的方法，查找方法上是否有@RequestMapping注解，如果存在，则将@RequestMapping中的信息保存到RequestMappingInfo中

（4）将方法返回的RequestMappingInfo和bean的RequestMappingInfo结合起来，并保存为Map<Method, RequestMappingInfo>集合

（5）对每个method创建HandlerMethod类，并将RequestMappingInfo -> HandlerMethod保存到mappingRegistry的属性mappingLookup中

（6） 将url -> RequestMappingInfo 的映射保存到mappingRegistry的属性urlLookup中

## @RequestMapping --- 请求转发

​	上述部分主要讲述了，Springboot在启动的时候，RequestMappingHandlerMapping是如何解析被@RequestMapping修饰的类或者方法的，在本小节中，将带领大家了解一下http请求是如何被正确的路由到对应的方法中的。

​	我们知道SpringMVC中流量转发是通过DispatcherServlet类的doService方法中的doDispatch方法实现的，下面我们一起来看下doDispatch的源码。

![doDispatch方法部分源码](image-20220123200238042.png)

​	我们只关注doDispatch中部分相关源码。当http请求到来时，doDispatch会通过getHandler方法获取Request对应的Handler，此处的Handler就是我们自己编写的方法，所以我们只需要继续看getHandler的源码即可。

![getHandler源码](image-20220123201008364.png)

​	getHandler函数的源码比较简单，就是通过遍历handlerMappings来获取Handler的执行链。我们从图中可以看到总共有7个handlerMappings，在前面小节中我们已经知道了RequestMappingHandlerMapping才是解析的关键，所以我们继续深入查看RequestMappingHandlerMapping中getHandler的源码。

![RequestMappingHandlerMapping中getHandler源码](image-20220123201411809.png)

从图中可以看到getHandler调用了getHandlerInternal方法来获取handler，我们继续深入探索。

![getHandlerInternal源码](image-20220123201623903.png)

图中红框的代码为关键代码，调用lookupHandlerMethod来获取指定lookupPath的handlerMethod，我们继续查看lookupHandlerMethod源码。

![lookupHandlerMethod源码](image-20220123202048571.png)

​	lookupHandlerMethod函数的作用就是通过给定的lookupPath，从mappingRegistry中获取HandlerMethod的最佳匹配，在我们实际的开发中，一个url可能不仅仅匹配到一个RequestMappingInfo，所以该方法会帮我们返回一个最佳的匹配。这部分的具体讲解请看文章XXX。
