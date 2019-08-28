## Spring 注解驱动开发(四) 对组件进行赋值

在Spring中，对组件进行赋值的方式有如下几种：

1. 使用`@Value`注解
2. 使用`@PropertySource`或`@PropertySources`向运行环境中导入属性值
3. 使用`@Profile`，根据当前的环境，动态地激活或切换一系列组件
4. 使用`@Autowire`实现自动装配

---

### `@Value`

对于普通Bean中的一些属性，可以使用`@Value`注解对其进行赋值，这个注解需要设置一个值，这个值可以是：

1. 基本的数据，字符串等
2. Spring的表达式：SpEL，#{}
3. 使用${}取出配置文件(*.properties)中的值，也就是运行环境中的值(context中的值)

举例如下：

```java
@Value("lma") // 将字符串'lma'赋值给name属性
private String name;

@Value("#{27-2}") // 使用SpEL表达式，将计算后的结果，即25赋值给age属性
private Integer age;

// 使用${}表达式，从运行环境中将key为person.nickName的值赋给nickName属性
// 这些配置文件中值，一般定义在*.properties文件中
// *.properties文件，通过@PropertySource注解可以加载到运行的Spring环境(Context)中
@Value("${person.nickName}") 
private String nickName;
```

#### #{} 与 ${} 的区别

这两种取值方式都可以配合`@Value`注解对某些属性进行赋值，但两者又有一些稍微的不同。

首先 `#{}` 是SpEL表达式，通常用于获取Spring容器中某一个Bean的属性，或者调用某个Bean的方法，当然也可以表达常量；

其次 `${}` 可以获取对应属性文件中定义的属性值，常用于获取properties文件中的值 。

```java
@Data
public class Father {
    @Value("${person.firstName}")
    private String firstName;
    @Value("${person.lastName}") // 从属性文件中获取值，只能使用${}；如果使用#{}，会报错
    private String lastName;
}

@Data
public class Child {
    // 使用#{}获取容器中名为father的Bean，然后将该Bean的firstName的值赋给当前的属性
    @Value("#{father.firstName}") 
    private String firstName;
    @Value("XiaoJun") // 基本数据类型
    private String lastName;
}

@Configuration // 配置类
@PropertySource("classpath:/person.properties")
public class MainConfig {
    @Bean
    public Father father(){ return new Father(); }
    @Bean
    public Child child(){ return new Child(); }
}

// 测试类
public class Test {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext =
                new AnnotationConfigApplicationContext(MainConfig.class);
        Father father = applicationContext.getBean(Father.class);
        Child child = applicationContext.getBean(Child.class);
        System.out.println(father.toString());
        System.out.println(child.toString());
        applicationContext.close();
    }
}

// 运行结果：
// Father(firstName=Ma, lastName=Long)
// Child(firstName=Ma, lastName=XiaoJun)
```

---

### `@PropertySource `和 `@PropertySources`

使用`@PropertySource` 或  `@PropertySources`来加载外部的配置文件，将配置文件中的属性以key ：value的形式加载到运行环境中(environment of application context)，然后在需要的位置使用 ${} 的形式去取出需要的值。

```java
@Configuration
@PropertySource(value = {"classpath:/person.properties"}) 
@PropertySources(value = {
        @PropertySource(value = {""}),
        @PropertySource(value = {""}),
        @PropertySource(value = {""})
})
public class MainConfigPropertyValue {}
```

---

### `@Profile`

`@Profile`注解，由Spring提供，可以根据当前的环境，动态地激活或切换一系列组件。

`@Profile` 可以指定组件在哪种情况下才能被注册到容器中，不指定的情况下，默认任何情况下所有组件都可以注册到IOC容器中；

- 加了@Profile注解的bean，只有在指定的环境下才能加载到IOC容器中。默认环境是：default
- 加了@Profile注解的配置类，只有指定的环境下，整个配置类的所有配置才能生效
- 没有加@Profile注解的bean，在任何环境下都是默认加载的

```java
@Profile({"test"}) // 该配置类，在test环境下激活
@PropertySource({"classpath:/person.properties"})
@Configuration
public class MainConfigOfProfile {
    @Bean
    public Car car() {return new Car();}
    
    @Bean
    public Father father() {return new Father();}

    @Bean
    @Profile({"prod"}) // 该Bean在prod环境下配置
    public Child child() {return new Child();}
}

public class MainConfigOfProfileTest {
    /**
     * 方式 1> 使用命令行参数的方式(在虚拟机参数位置): -Dspring.profiles.active=test
     * 方式 2> 使用代码的方式激活某种环境
     */
    public static void main(String[] args) {
        // 1. 创建applicationContext环境
        AnnotationConfigApplicationContext context = 
            new AnnotationConfigApplicationContext();
        // 2. 设置需要激活的环境
        context.getEnvironment().setActiveProfiles("test");
        // 3. 注册配置类
        context.register(MainConfigOfProfile.class);
        // 4. 启动刷新容器
        context.refresh();

        String[] names = context.getBeanDefinitionNames();
        for (String name : names) {
            System.out.println(name);
        }
        context.close();
    }
}

// 打印结果
// org.springframework.context.annotation.internalConfigurationAnnotationProcessor
// org.springframework.context.annotation.internalAutowiredAnnotationProcessor
// org.springframework.context.annotation.internalCommonAnnotationProcessor
// org.springframework.context.event.internalEventListenerProcessor
// org.springframework.context.event.internalEventListenerFactory
// mainConfigOfProfile 配置文件类
// car 名字为car的bean
// father 名字为father的bean
// 没有名字为child的bean，因为这个bean的激活环境和设置的激活环境不一致
```

### `@Autowire`

使用`@Autowire`可以进行自动注入，该注解可以标注在构造器、方法参数、方法、属性位置，无论标注在什么位置都可以从IOC容器中获取组件的参数。

默认地，优先按照类型去容器中寻找相应的组件，找到后就去赋值；如果找到了多个相同类型的组件，再根据属性的名称作为id去容器中寻找，可以在`@Autowire`注解上再使用`@Qualifier`来指定需要装配的组件的id，而不是优先使用属性名。

同时，默认情况下，Spring一定要将所有的属性赋好值，如果发现某个组件在容器中没有(No such bean)，就会报错。`@Autowire`注解的required=true，表示必须装配，可以将修改为required=false，此时装配就不是必须的，如果No such bean，也不会报错。

在`@Autowire`注解上也可以使用`@Primary`来指定自动装配时，默认首选的Bean。

这里就涉及了一个优先级的问题，当`@Primary` 和 `@Qualifier` 同时存在时，谁最终起作用。答案是：`@Qualifier`注解最终起作用。

```java
@Qualifier("bookDao") // 指定使用id为bookDao的Bean来进行装配
@Autowired
private BookDao bookDao; // 在需要的位置进行注入

@Primary // 设置bookDao2的优先级
@Bean("bookDao2")
public BookDao bookDao() {
    BookDao bookDao = new BookDao();
    bookDao.setLabel("Manner value");
    return bookDao;
}

// 上面只是两个代码片段，有两个类型为BookDao的Bean注册到了IOC容器中
// bookDao2被@Primary修饰，bookDao被@Qualifier修饰
// 最后生效的是bookDao，而不是bookDao2
```

同时，Spring还支持使用`@Resource`(JSR250)和`@Inject`(JSR330)，这两个注解都是Java规范中定义的。

#### `@Resource`

@Resource 可以和 @Autowired 一样实现自动装配功能， 默认是按照组件的名称进行装配，也可以通过`@Resource(name = "bookDao")`去指定注入的bean，但是@Resource不支持@Primary，也不支持`@Autowired(required = false)`的方式。

#### `@Inject`

@Inject: 需要导入`javax.Inject`包，和@Autowired的功能一样支持@Primary，但是@Inject不支持`@Autowired(required = false)`的方式。

总结一下，知道Spring博取众长就行了，最终使用的还是`@Autowire`，但是见到别人使用另外这两个注解也不要大惊小怪。

---

最后，我们可以想一下，Spring是如何做到对于组件的赋值呢？当我们标注一个`@Value`或`@Autowire`时，Spring怎么将这些值设置到我们需要的位置呢？

还是依赖于BeanPostProcessor机制，Spring实现了一个类`AutowiredAnnotationBeanPostProcessor`，专门用来处理组件的赋值，我们可以先简单地来看下这个类的Java doc：

```java
// {@link org.springframework.beans.factory.config.BeanPostProcessor} implementation
// that autowires annotated fields, setter methods and arbitrary config methods.
// Such members to be injected are detected through a Java 5 annotation: by default,
// Spring's {@link Autowired @Autowired} and {@link Value @Value} annotations.
```



