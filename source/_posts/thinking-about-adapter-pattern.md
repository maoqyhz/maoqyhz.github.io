---
title: 谈谈适配器模式
tags:
  - 设计模式
  - Java
categories:
  - 技术
copyright: true
date: 2019-09-08 21:19:59
---

适配器模式 (Adapter Pattern)：将一个接口转换成客户希望的另一个接口，使接口不兼容的那些类可以一起工作，其别名为包装器 (Wrapper)。适配器模式既可以作为类结构型模式，也可以作为对象结构型模式。

<!-- more -->

![UML](adapter.png)

设计模式的目的本身应该是使得软件设计更具合理性，松耦合，易于扩展。我们在学习的过程中，无论是书上还是博客上，一般都会描述模式的定义、UML 图以及基本的实现方式。但是在我看来，在模式的实现和运用上似乎并不需要完全按照模式本身的结构来。

在 SpringMVC 中适配器模式主要用于执行目标 Controller 中的请求处理方法。

看了一些资料，在这里感觉使用 Adapter 的目的，无非是减少 if 的使用，并且减少大量的请求处理代码和 DispatcherServlet 之前的耦合。因为不同的 Handler 有不同名称的请求处理方法，如果不使用统一接口的 Adapter，势必会添加一些硬编码的类型判断（Controller 和 Servlet 都是 Handler 的一种）。因此封装一系列的 Adapter 与各个类型的 Handler 对应，在 DispatcherServlet 中就可以直接通过 Handler 寻找合适的 Adapter。当然，Adapter 列表会在 Bean 初始化的过程中预先加载好，所以本质上还是遍历匹配类型，但是在写法和设计上和直接使用一组 I if 语句判断有一定的区别。

在 JPA 中也是如此， JpaVendorAdapter 就是一个适配器，不同的 JPA 提供者需要去实现不同的 Adapter。实际上就是一个标准化的过程，这两个例子所表现的都是一致的。虽然有很多的处理者，但是都有统一的方法接入。例如，HandlerAdapter 的 handle() 方法。最典型的插座的例子也是，无论输入的电压是 220v 还是 110v，但我最终需要的就是 5v，这个就需要适配器来进行转换。

设计模式就好比张三丰手里的太极剑，不拘泥于招式，用意不用力，在日后开发过程中还需慢慢体会。

## Reference

- [浅谈适配器设计模式](https://zhuanlan.zhihu.com/p/56518978)
- [设计模式 | 适配器模式及典型应用](https://blog.csdn.net/wwwdc1012/article/details/82780560)
- [SpringMVC 从 request 到 controller 过程详解](https://blog.csdn.net/zgzczzw/article/details/53926635)

