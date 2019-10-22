# Spring or Springboot Run with or without External Tomcat

> Java Web 端多多少少总会和 Spring framework 或 Spring boot 产生关联。Spring 相关的技术并不是必须的，但是一个 Java Web 的应用必须要一个运行时的容器，也就是动态 Web 服务器，常用服务器的有 Tomcat、Jetty、Undertow。以 Tomcat 为例，我们来考虑一下，如果使用了 Spring 相关的技术，这两个容器(Spring 容器和 Tomcat 容器)是如何启动的，它们又是怎样一种关系。

> 一般而言，Java Web 应用程序需要一个 web.xml 文件，在这个文件中描述了当前 Web 应用部署的情况，如果这个 Web 应用不包含任何的 servlet、filter、listener 组件，即这是一个静态的 Web 应用，那么就可以不需要这个 web.xml 文件。换句话来说，只包含了静态文件的应用并不需要这个文件。

> 如果是一个动态的 Web 应用呢？是否必须包含这个 web.xml 文件呢？答案是否定的。因为在 `servlet3.0` 及其以后的规范中，引入了一个新的特性 --- 共享库/运行时的可插拔特性，基于这一新特性，再配合注解的方式，可以完全替代该 web.xml 配置文件。

> 同时，如果 Java Web 应用被打成了一个 war 包，我们可以将该 war 包放置在一个外部的动态 Web 服务器中，如 Tomcat，随着 Tomcat 的启动，就可以访问该应用了。而 Spring boot 最终将应用打包为一个 jar 包，通过命令行的方式来启动该 Java Web 应用，那么此时的 Web 动态服务容器，就不是外部手动配置的容器了。那 Spring boot 又是如何处理 Web 的动态服务呢？其实是 Spring boot 内置了一个 Web 动态服务器，随着 Spring boot 的启动，也会初始化和运行该内嵌的服务器。那此时就存在着两个容器的启动顺序与激活过程，带着这样的思考，我们来考虑一下几个问题。



## 包含 web.xml 的 Java Web 应用

一般而言，Java web 项目都会包含一个 web.xml 的配置文件，通常放置在 WEB-INF 目录中。如下图所示，这是 Tomcat 服务器中自带的一个示例工程，如果打开这个 web.xml 文件，可以看见其中定义了该 web 工程的一些描述信息，包含了 servlet 规范中的三大组件，分别是 servlet、filter、listener。

![1571663049360](./imgs/webapp-example.png)

在了解了基本的 Java web 项目后，我们来考虑一下：如果我们的项目是基于 SpringMVC 开发，最终项目是如何启动的？

![1571663452433](./imgs/javaweb-springmvc.png)

如上图所示的，在 web.xml 中配置了 Spring 和 SpringMVC。当项目打成 war 包并部署在 Tomcat 服务器后，当服务器启动时，根据配置文件中信息，Spring 会根据类路径下的 applicationContext.xml 文件进行初始化，这一初始化的开始是由一个 Spring 实现的 listener 来帮助完成的。同时 SpringMVC 的核心控制器也以 servlet 的方式进行了注册，这也就意味者 Tomcat 启动时会初始化这个前端控制器。

由此，可以得知，当我们在 web.xml 中将 Spring 以及 SpringMVC 的信息配置好，当 Tomcat 启动时，会去拉起 Spring 容器的初始化。简言之，Tomcat 驱动 Spring。



## 采用注解的 Java Web 应用

在 Servlet3.0 及以上的规范中，引入了一个重要的新特性：共享库/运行时的可插拔特性。这一特性是这样描述的：

1. 对于每一个应用而言，当应用启动时，由 Web 容器(如 Tomcat )创建一个 ServletContainerInitializer 实例。如果某个框架(如 Spring )提供了 ServletContainerInitializer的实现，Web 容器也会实例化框架提供的这个实例。
2. 但是框架提供的实现，必须绑定在 META-INF/services 目录中的一个叫做javax.servlet.ServletContainerInitializer 的文件中。
3. 除了这种方式外，还可以使用 @HandlesTypes 注解，来注入一些我们感兴趣的类。

在这一新特性的基础上，我们可以基于 Java Config 的方式来完成 Spring 和 SpringMVC 容器的配置，一般 Spring 和 SpringMVC 会配置成一对父子容器( Spring 官方推荐)。

```java
/**
 * web 容器在启动创建对象的时候，就会调用该类的方法来初始化 Spring 容器和前端控制器
 * @author lma
 */
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    /**
     * 获取根容器的配置类
     * 相当于以前Spring的配置文件
     * 用于创建根容器(父容器)
     *
     * @return
     */
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class[]{RootConfig.class};
    }

    /**
     * 获取web容器的配置类
     * 相当于以前的SpringMVC的配置文件
     * 用于创建web容器(子容器)
     *
     * @return
     */
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[]{AppConfig.class};
    }
    
    /**
     * 获取DispatcherServlet的映射信息
     * / : 拦截所有请求(包括静态资源，xxx.js，xxx.png ..)，但不包括 *.jsp
     * /* : 拦截所有请求，包括 *.jsp
     * jsp页面是tomcat的jsp引擎解析
     *
     * @return
     */
    @Override
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }
}
```

当我们自定义一个类 `MyWebAppInitializer`，并继承了 `AbstractAnnotationConfigDispatcherServletInitializer` 后，可以覆写这个类中的方法，在这些方法中，可以以 Java 代码的方式来配置 Spring 的容器。为什么继承了这个 Spring 提供的类以后，Tomcat 容器启动时就可以初始化 Spring 容器和前端控制器。

![1571664589129](./imgs/springweb-info.png)

在 spring-web:xxx.RELEASE.jar 的 META-INF/services 的 javax.servlet.ServletContainerInitializer 文件中有且仅有这个配置类，在这个 `SpringServletContainerInitializer` 类上，有一个 `@HandlesTypes(WebApplicationInitializer.class)` 注解。

![1571664905643](./imgs/SpringServletContainerInitializer.png)

这个注解正是 servlet3.0 中提到的 `@HandlesTypes`，通过这个注解可以将 `WebApplicationInitializer` 的实例进行注册。接着来下看 `WebApplicationInitializer` 的继承关系：

![1571665061448](./imgs/WebApplicationInitializer.png)

现在，我们能够看到整体的执行流程。并且能够清晰地看出，Tomcat 7.0 本版实现了 servlet3.0 规范以后，web.xml 不再是必须品。当我们需要在 Tomcat 容器启动的时候，激活、初始化或实例化一些必要的类时，不再是去 web.xml 中进行配置，而是以 Java config 的方式来实现 --- 在 jar 的 META-INF/services/javax.servlet.ServletContainerInitializer 文件中，以全类名的方式进行配置。就是上面配置 Spring 和 SpringMVC 容器那样。

此时，最后来考虑一下，启动的顺序又是如何？没错，还是 Tomcat 驱动 Spring。

 

## Spring boot 激活内置的 Web 服务器

Spring boot 是基于 Spring framework 的一款快速开发框架，核心理念是约定大于配置，也就是说，以前我们进行的很多配置，在使用 Spring boot 后，会被框架所处理，即不需要我们显示的去配置，遵循框架的约定，就可以快速地开发一个服务。

当我们使用了 Spring boot 后，可以将项目打成一个 jar 包，然后通过 Java 命令来启动一个服务，即使是一个 web 服务，也可以直接以 java -jar 的形式来启动，跳过了部署到外部 Tomcat 服务器的过程。但我们需要知道的是，基于 servlet 规范开发的 Java web 服务，是必须依赖于一个 servlet 容器，也就是 Tomcat 是必备的。那么 Spring boot 是如何处理简化这一过程的？Spring boot 中内嵌了一个 servlet 容器，当前支持 Tomcat、Jetty 和 Undertow，默认使用 Tomcat。当 Spring boot 推断出当前的项目是一个 web 工程(如何推断？判断当前的 Spring Context 中是否包含一个前端控制器，即：DispatcherServlet)时，在 Spring boot 启动过程中，会创建一个 servlet 容器的实例。

在理解了相互的调用关系后，可以尝试深入 Spring boot 的世界，具体地来看看，一个 SpringApplication 是如何跑起来的。详细的文档，可以参看[《SpringApplication to Fire in detail》](./spring-application-to-fire-in-detail.pdf)。Spring boot 的 run() 方法使用了一个典型模板方法的设计模式，其中定义了SpringApplication 的运行步骤。在这些步骤中，有一个极其重要，refreshContext(context)，正是在这一步中，Spring boot 调用了 Spring framework 的 refresh() 方法。这个 refresh() 方法，又是一个重量级的方法，它定义了Spring容器的初始化和创建过程，关于这个有兴趣的可以参看 [《Spring 容器的初始化和创建过程》](./spring-container-initial-and-create.pdf)。

在这个 refresh() 方法中，有如下一个 onRefresh() 方法，从注释文档中可以看出：它用于在特殊的上下文环境子类中初始化一些特殊的 Bean 。进入这个方法，会发现这是一个抽象方法，没有任何的实现，这意味着在一个普通的Spring环境中，这一步骤可以直接跳过，或者我们可以在这里定义一些逻辑。

![1571711193435](./imgs/spring-on-refresh.png)

但是，在一个 Spring 的 Web 环境中，有一个类实现了这个方法。从这个方法中，可以直接地看到了 createWebServer()，终于追到了尽头，正是在这个方法中，Spring 框架初始化了一个 Tomcat (或其他的内嵌Web容器)，并启动了 Tomcat。

![1571711283188](./imgs/create-web-server.png)

最后，再来重复考虑一下，调用顺序又是如何？Spring boot 驱动 Tomcat。



## Spring boot 在外部的 Web 服务器中运行

当我们将一个 Sping boot 的 web 应用打包为一个 war 包时，将其放置在一个外部的 Tomcat 容器的 webapp 目录下，随着我们启动 Tomcat 容器，Spring boot 应用也会被激活。这又是怎么做到的呢？

一般想要达到这样的效果，需要做一些额外的配置，即必须编写一个 `SpringBootServletInitializer` 的子类，并复写其 configure() 方法。

![1571712321255](./imgs/spring-boot-servlet-initializer-configure.png)

其中，`SpringBootServletInitializer` 实现了 `WebApplicationInitializer` 接口。

![1571712415433](./imgs/spring-boot-servlet-initializer.png)

似曾相识，这个 `WebApplicationInitializer` 接口在第二节中就已经见过。从上面的描述中，我们知道一个 Tomcat 容器( Web服务器 )依据 Servlet3.0 中相应的规范，会扫描到`org.springframework.web.SpringServletContainerInitializer` 这个类，它使用了 `@HandlesTypes` 注解标注，即 `WebApplicationInitializer` 的子类都会得到实例化。

所以 SpringBootServletInitializer 会被实例化，并会执行它的 onStartup() 放在，在这个方法中会创建 Spring 的容器。

![1571712837495](./imgs/spring-boot-servlet-initializer-onstartup.png)

并且在 createRootApplicationContext() 方法中，会调用 configure(builder) 方法。

![1571713193429](./imgs/spring-boot-servlet-initializer-create-root-application-context.png)

而 configure(builder) 这个方法被我们自己实现类中的方法覆写了，所以我们自己的configure(builder) 会执行，在这个方法执行的过程中，将该应用的启动程序传入。然后在后面的步骤中，激活了 Spring boot 容器。

最后，考虑这种情形情形下，调用顺序又是如何？Tomcat 驱动 Spring boot。



> Spring boot 的出现，极大的简化了开发。这种简化并不代表着轻松或容易掌握，反而由于 Spring boot 对一些底层支持的封装屏蔽，容易产生一种云深不知处的困惑。对于读者，我自己感觉总是需要找到 Spring 与底层契合的点，往往这些契合的点，会给我们理解框架带来一些启示，例如 Spring Web 与 Servlet 3.0 新特性的契合。



> @date 2019/10/19
>
> @update 2019/10/22