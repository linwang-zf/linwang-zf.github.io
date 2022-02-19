---
title: SpringMVC系列(三)---参数解析流程
author: linWang
date: 2022-02-19 18:02:33
tags: SpringMVC系列
categories: Spring源码
top: true
---

>   本章详细的介绍了@RequestBody、@RequestParam、@PathVariable的参数解析原理。

<!--more-->

在之前的文章中我们介绍了@RequestMapping注解的原理和uri匹配规则的原理，感兴趣的小伙伴可以点击下面的链接访问。

*   [SpringMVC系列(一)---RequestMapping注解的原理](https://blog.linwang.tech/2022/01/23/SpringMVC%E7%B3%BB%E5%88%97(%E4%B8%80)---RequestMapping%E6%B3%A8%E8%A7%A3%E7%9A%84%E5%8E%9F%E7%90%86/)

- [SpringMVC系列(二)---RequestMapping中Url匹配原理](https://blog.linwang.tech/2022/02/19/SpringMVC%E7%B3%BB%E5%88%97(%E4%BA%8C)---RequestMapping%E4%B8%ADUrl%E5%8C%B9%E9%85%8D%E5%8E%9F%E7%90%86/#more)

SpringMVC将Request成功匹配到某个指定的HandlerMethod后，那又是如何进行参数解析的呢，下面将为大家详细的介绍下参数解析的原理。
在开发的过程中，为了能够获取到url中的参数，我们常用的注解有：@RequestBody，@RequestParam，@PathVariable等，下面我们对这三个注解进行一个介绍。

## @RequestBody注解原理
![](1.png)

![](2.png)

我们将以上述的例子来讲解RequestBody是如何将url中的参数解析到TestDemoAO类中的。

![](3.png)

跳过前面url的匹配规则，不再赘述，直接来到通过RequestMappingHandlerAdapter调用handleMethod方法，看下handle的源码。

![](4.png)

在handler方法中调用了invokeHandlerMethod方法，经过我们层层深入，终于找到了参数解析的地方，在InvocableHandlerMethod类中。

![](5.png)

下面我们来一起看下InvocableHandlerMethod类中的getMethodArgumentValues函数的源码。

![](6.png)

getMethodArgumentValues方法一共做了如下事情：

- 获取HandlerMethod的所有参数
- 遍历每个参数，从参数解析器列表中获取能够解析当前参数的resolver
- 调用解析器来解析当前参数，并获取参数的值
- 返回参数值列表

从上述步骤中，可以看到对于不同类型的参数，需要的解析器也不同。对于本例中含有@RequestBody注解的参数，对应的解析器为RequestResponseBodyMethodProcessor。

![](7.png)

上述为RequestResponseBodyMethodProcessor解析器解析参数的源码，红框中的函数为解析的核心逻辑。

![](8.png)

上图中的代码为readWithMessageConverters中的核心代码。通过遍历messageConverters获取适当的convert，并判断canRead，在RequestBody中，只有content-type为application/json时，canRead才会返回true，其余均返回false。看下其源码。

![](9.png)

从上图可以看出，request中的mediaType必须为application/json时，canRead返回true，此时才会去获取Request中的body并解析为Java的实体对象。

![](10.png)

之后调用父类AbstractJackson2HttpMessageConverter中的read方法将inputMessage中的body解析为javaType。如下图

![](17.png)

![](11.png)

上述中ObjectMapper.readValue是jackson-databind包中的方法，将一个指定的inputStream解析为javaType类型，此处不再继续深入。通过上述步骤则将request请求中的body解析为了我们的TestDemoAO实体，然后通过反射继续调用HandlerMethod方法。

## @PathVariable注解原理

在RequestMapping的url匹配规则文章中，我们知道了如果url匹配规则中包含{param}参数，MVC会将参数名作为key，参数值作为value存在request的attribute中，具体详解请看XXX文章。
我们可以大胆的猜想，@PathVariable注解的参数一定是从request的attribute中通过key来获取到这个参数值，并赋值给方法的参数的，下面我们一起来验证下这个猜想是否正确。
调用的过程与@RequestBody的完全一直，我们不再赘述。两者的区别在于通过参数获取的解析器不一致，被@PathVariable修饰的参数对应的解析器为PathVariableMethodArgumentResolver类。

![](12.png)

该解析调用其父类AbstractNameValueMethodArgumentResolver的resolveArgument方法，其中主要逻辑在红框中的resolveName方法中。

![](13.png)

上述图中可以验证我们的猜想是正确的，该方法会从request的attribute中拿到保存所有参数的Map，并从中通过key获取参数值并返回。

## @RequestParam注解原理

在有了上述两个注解的学习经验后，我们这次直接来到获取解析器的源码位置。

![](14.png)

从上图中可以看出@RequestParam修饰的参数对应的解析器为RequestParamMethodArgumentResolver。继续看该解析器的resolveArgument方法中的resolveName方法，在该方法中找到如下代码：

![](15.png)

从上图中可以发现RequestParamMethodArgumentResolver解析器从request的Paramter中查询指定name的value值，并返回。

## 总结

- 不管是RequestBody、PathVariable还是RequestParam注解，本质的区别就在于根据参数获取的解析器不同
    - RequestBody修饰的参数  ->   RequestResponseBodyMethodProcessor
    - PathVariable修饰的参数   ->   PathVariableMethodArgumentResolver
    - RequestParam修饰的参数 ->  RequestParamMethodArgumentResolver
    - 未加任何注解的参数   ->  RequestParamMethodArgumentResolver(这也是为啥我们可以省略RequestParam注解的原因）

下面附上这几个解析器的类图。

![](16.png)
