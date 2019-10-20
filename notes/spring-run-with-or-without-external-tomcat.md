# Spring or Springboot Run with or without External Tomcat

> Java Web 端多多少少总会和 Spring framework 或 Spring boot 产生关联。Spring 相关的技术并不是必须的，但是一个 Java Web 的应用必须要一个运行时的容器，也就是动态 Web 服务器，常用服务器的有 Tomcat、Jetty、Undertow。以 Tomcat 为例，我们来考虑一下，如果使用了 Spring 相关的技术，这两个容器(Spring 容器和 Tomcat 容器)是如何启动的，它们又是怎样一种关系。

> 一般而言，Java Web 应用程序需要一个 web.xml 文件，在这个文件中描述了当前 Web 应用部署的情况，如果这个 Web 应用不包含任何的 servlet、filter、listener 组件，即这是一个静态的 Web 应用，那么就可以不需要这个 web.xml 文件。换句话来说，只包含了静态文件的应用并不需要这个文件。

> 如果是一个动态的 Web 应用呢？是否必须包含这个 web.xml 文件呢？答案是否定的。因为在 `servlet3.0`及其以后的规范中，引入了一个新的特性 --- 共享库/运行时的可插拔特性，基于这一新特性，再配合注解的方式，可以完全替代该  web.xml 配置文件。

> 同时，如果 Java Web 应用被打成了一个 war 包，我们可以将该 war 包放置在一个外部的动态 Web 服务器中，如 Tomcat，随着 Tomcat 的启动，就可以访问该应用了。而 Spring boot 最终将应用打包为一个 jar 包，通过命令行的方式来启动该 Java Web 应用，那么此时的 Web 动态服务容器，就不是外部手动配置的容器了。那 Spring boot 又是如何处理 Web 的动态服务呢？其实是 Spring boot 内置了一个 Web 动态服务器，随着 Spring boot 的启动，也会初始化和运行该内嵌的服务器。那此时就存在着两个容器的启动顺序与激活过程。带着这样的思考，我们来考虑一下几个问题



## 包含 web.xml 的 Java Web 应用

包含了 web.xml 文件的 Java Web 应用



## 采用注解的 Java Web 应用

使用注解的方式来替代 web.xml 文件



## Spring boot 激活内置的 Web 服务器

Spring boot 以 jar 包的形式启动，在启动的过程中，激活 Web 服务器



## Spring boot 在外部的 Web 服务器中运行

Spring boot 项目打为war包，使用外部的 Tomcat 容器，此时外部的 Tomcat 又如何激活Spring boot?