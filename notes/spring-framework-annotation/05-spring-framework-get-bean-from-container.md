## Spring 注解驱动开发(五) 从IOC容器中获取组件

当我们需要容器中的一个组件时，我们可以通过`applicationContext`去获取，获取的方式可以是该组件的类文件、该组件名，或者可以通过一个方法获取容器中所有的组件。

通过下面的例子，我们先来简单的看下，如何从容器中获取一个组件：

```java
public class TestAware {
    @Test
    public void testOne() {
        AnnotationConfigApplicationContext applicationContext = 
            new AnnotationConfigApplicationContext(MainConfig.class);

        // 根据类文件获取一个Bean
        Jasmine jasmine = applicationContext.getBean(Jasmine.class);
        Daffodil daffodil = applicationContext.getBean(Daffodil.class);
        
        // 获取容器中注册Bean的数量
        int count = applicationContext.getBeanDefinitionCount();
        // 获取所有Bean的名称
        String[] beanDefinitionNames = applicationContext.getBeanDefinitionNames();
        // 根据类文件获取所有Bean的名称
        String[] beanNamesForType = 
            applicationContext.getBeanNamesForType(Jasmine.class);
        // 根据类文件获取所有的Bean实例，封装在一个Map集合中
        Map<String, Daffodil> beansOfType = 
            applicationContext.getBeansOfType(Daffodil.class);
        // 根据注解类型获取所有的Bean实例，封装在一个Map集合中
        Map<String, Object> beansWithAnnotation = 
            applicationContext.getBeansWithAnnotation(Controller.class);
        
        applicationContext.close();
    }
}
```

上面的这种方式是我们以编程的方式，人为的从IOC容器中获取某些Bean的过程，其实在使用Spring的过程中，有时候也会以声明的方式去从容器中获取某些组件，如，当我们使用`@Autowire`注解为某一个属性赋值的时候，都先从容器中获取了必须的组件，然后再对该组件进行赋值。

日常的编程中，我们使用注解(声明)的方式较多，直接以编码的方式从容器中获取组件或者服务的方式并不多。

接下来，来说一个话题 --- Spring Aware

### Spring Aware

我们知道Spring最大的亮点就是控制反转(IOC)和依赖注入(DI)，这两者没有什么本质上的区别。IOC或DI容器中，我们自己所有的Bean对Spring容器的存在是没有意识的，即我们可以将Spring的容器更换为其他的容器，如Google Guice，这是因为Bean与IOC容器的耦合度是很低的。

但在实际开发中，不可避免地需要用到Spring容器本身的功能资源，这时我们的Bean就必须要意识到容器的存在，才能调用Spring所提供的资源，这就是所谓的Spring Aware。其实Spring Aware本来就是Spring设计给内部框架使用的，如果使用了Spring Aware，我们自己的Bean就会意识到容器的存在，此时Bean也将会和IOC耦合在一起。

Spring提供了如下的Aware接口：

| 接口名                         | 功能                                                  |
| ------------------------------ | ----------------------------------------------------- |
| BeanFactoryAware               | 获取当前bean factory，这样可以调用容器的服务          |
| BeanNameAware                  | 获得到容器中Bean的名称                                |
| ApplicationContextAware        | 获取当前的application context，这样可以调用容器的服务 |
| MessageSourceAware             | 获取message source，这样可以获取文本信息              |
| ApplicationEventPublisherAware | 应用事件发布器，可以发布事件                          |
| ResourceLoaderAware            | 获得资源加载器，可以获得外部资源                      |

Spring Aware的目的是为了让Bean获得Spring容器的服务。因为ApplicationContext接口集成了MessageSource接口、ApplicationEventPublisher接口和ResourceLoader接口，所以Bean实现了ApplicationContextAware就可以获得Spring容器的所有服务，但原则上我们还是用到什么服务，就实现什么接口。

下面，我们来看一个例子：

```java
@Data
public class Jasmine implements BeanNameAware {
    private String id;
    private String name;
    private Integer num;
    // setBeanName方法来自于BeanNameAware接口
    public void setBeanName(String str) {this.id = str;}
}

@Data
public class Daffodil {
    private String id;
    private String name;
    private Integer num;
}

@Configuration // 配置文件
public class MainConfigOfAware {
    @Bean("myJasmine") // 设置注入容器中Bean的名称
    public Jasmine jasmine() {
        Jasmine jasmine = new Jasmine();
        jasmine.setName("jasmine");
        jasmine.setNum(10);
        return jasmine;
    }

    @Bean // 默认Bean的名称和方法名一致
    public Daffodil daffodil() {
        Daffodil daffodil = new Daffodil();
        daffodil.setName("daffodil");
        daffodil.setNum(11);
        return daffodil;
    }
}

// 测试类
public class TestAware {
    @Test
    public void testAware() {
        AnnotationConfigApplicationContext applicationContext = 
            new AnnotationConfigApplicationContext(MainConfigOfAware.class);
        Jasmine jasmine = applicationContext.getBean(Jasmine.class);
        Daffodil daffodil = applicationContext.getBean(Daffodil.class);
        System.out.println(jasmine);
        System.out.println(daffodil);
        applicationContext.close();
    }
}

// 测试结果
// Jasmine(id=myJasmine, name=jasmine, num=10) --- Jasmine：茉莉花
// Daffodil(id=null, name=daffodil, num=11) --- Daffodil：水仙花
// 我们在配置文件中，都没有为这两个Bean设置id，但结果中Jasmine有id，并且是容器中Bean的名称myJasmine，这是因为，Jasmine类实现了BeanNameAware接口，在接口的方法中，获取到了Bean的名称，同时将该名称赋值给了id
```

我们可能会想，Spring是如何实现这种Aware的功能的呢？

让我们来看一个类，在上面的描述中，知道了`ApplicationContextAware`功能比较强大，它理所当然地继承了Aware接口。与此同时，有一个类与`ApplicationContextAware`的名字很相似 --- `ApplicationContextAwareProcessor`，只是在后面加了一个Processor，我们再来看这个类，这个类实现了一个前面经常提及的接口 --- `BeanPostProcessor`，万剑归一的感觉。

再次感受了BPP机制地强大和广泛应用。

---

后记：由于文章的分节不是那么严谨，导致这一节东西比较少。但也接触了一个很重要Aware机制，其中Spring Aware这一部分基本上就是照着书上写的，感兴趣的可以去看原书 --- 汪云飞《Spring Boot 实战》

