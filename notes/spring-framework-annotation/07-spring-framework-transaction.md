## Spring 注解驱动开发(七) 声明式事务的原理

声明式事务是Spring中对数据库事务的简化处理，旨在于通过简单的注解配置就可以达到对于数据库不同级别的事务控制。

---

一般而言，使用注解的方式来开启Spring的声明式事务需要以下几个步骤：

1. 导入基本依赖：数据库驱动，数据源，SpringJDBC模块；
2. 配置数据源，JdbcTemplate(Spring提供的简化数据库操作的工具)；
3. 在配置类中使用@EnableTransactionManagement 开启基于注解的事务管理功能；
4. 配置事务管理器(PlatformTransactionManager)，来控制事务；
5. 在方法或者类中，需要事务支持的地方使用@Transactional标注；

---

通过注解来开启声明式事务的基本原理，与AOP的原来很相似，甚至可以理解为事务是简易版本的AOP。对于这部分的原理，可以通过以下几方面来理解：

1. 首先是从配置类中的`@EnableTransactionManagement`开始，这是一个组合注解；

   ```java
   @Target({ElementType.TYPE})
   @Retention(RetentionPolicy.RUNTIME)
   @Documented
   @Import({TransactionManagementConfigurationSelector.class})
   public @interface EnableTransactionManagement {
       boolean proxyTargetClass() default false;
       AdviceMode mode() default AdviceMode.PROXY;
       int order() default 2147483647;
   }
   ```

2. 其中最为重要的注解为@Import，通过@Import标签向IOC容器中注册了一个`TransactionManagementConfigurationSelector`的实例，这是一个典型的选择器。通过这个选择器，可以向容器中导入组件，由于在`@EnableTransactionManagement`定义了激活模式是`AdviceMode.PROXY`，所以在Selector中给组件导入的是2个组件：

   - `AutoProxyRegistrar`
   - `ProxyTransactionManagementConfiguration`

3. 在`AutoProxyRegistrar`中调用了`registerBeanDefinitions`方法，继续给容器中注册了`InfrastructureAdvisorAutoProxyCreator`组件，`InfrastructureAdvisorAutoProxyCreator`本质上也是一个后置处理器，这与实现AOP的`AnnotationAwareAspectJAutoProxyCreator`机制一样，利用后置处理器的机制在bean创建以后，对bean进行包装，最终向IOC容器中返回一个代理对象(包含了增强器)，代理对象执行方法时，利用链式调用机制进行递归调用。

4. 在`ProxyTransactionManagementConfiguration`中，通过`@Bean`向容器中注册了3个组件：

   - `BeanFactoryTransactionAttributeSourceAdvisor`  - advisor
   - `AnnotationTransactionAttributeSource`- source
   - `TransactionInterceptor` - interceptor上面的

   第一个是容器事务属性源拦截器，它需要依赖一个事务属性源和一个事务拦截器，分别是下面的两个组件 - source、interceptor。其中，source中包含了很多与事务解析相关的信息；interceptor中同时保存了事务属性源(`TransactionAttributeSource`)和父类中声明的事务管理器(`TransactionManager`)，同时`TransactionInterceptor`本质上是一个`MethodInterceptor`(方法拦截器)，所以在目标方法执行的时候，会调用到执行拦截器链。

   在事务处理中，这里的拦截器链就只有一个拦截器，即：`TransactionInterceptor`(事务拦截器)

5. 事务拦截器的工作机制：`TransactionAspectSupport#invokeWithinTransaction`

   - 先获取事务相关的属性，如果为空的话，表示这个方法没有事务

   - 再获取`PlatformTransactionManager`(平台事务管理器)

     如果事先没有添加指定任何`TransactionManager`，最终会从容器中根据类型获取一个`PlatformTransactionManager`；

     所以，我们也可以自己手动在配置类中以@Bean的方式向IOC容器中加入一个`PlatformTransactionManager`

   - 执行目标方法：

     如果发生异常，获取到事务管理器，利用事务管理器回滚本次操作；

     如果正常执行，同样获取到事务管理器，利用事务管理器提交本次操作。

---

Spring中的事务，与AOP相似，都是通过BBP的机制来实现的，即将需要增强的类或方法进行层层封装，最终在容器中产生一个经过代理增强的对象，这个对象中包含了整个方法的调用细节链。在具体方法真正执行时，其实是方法链的触发。



> 最近在看到一本技术书，其中讲到了Spring的理念，我们都知道Java是一门面向对象的语言，即面向Object编程，但Spring是面向Bean编程。Bean的本质是对Object的层层包装，所以能看出来，在Spring中最为重要的就是Bean了。在Spring的底层组件中，包含了三大组件：Context、Bean、Core，其中最为重要的当属Bean了，那其他两个组件又与Bean是怎样的关系呢？在书中，作者做了一个比喻，如果把Bean比作是一个演员，那么Context就是支撑演员表演的舞台，而Core就是演员的种种道具。同时，与其说是Core，不如说是Utils，因为Core中基本上是种种工具。
>
> 最后，我继续作者的比喻，当舞台搭好了，演员准备就绪了，道具也准备齐全了，即基础设施已经准备完毕了。但只有基础组件是不够的，演员到底表演什么节目呢？是芭蕾呢，还是秧歌呢？这就需要我们程序员作为导演来编排了，Spring已经编排了很多简单的基本舞步，如AOP、声明式事务等等，另外复杂的动作，需要我们利用好Spring的基础设施与基本舞步来搭建，Enjoy it !