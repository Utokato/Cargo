## Spring 注解驱动开发(二) 向IOC容器中添加组件

Spring提供了多种方式来支持我们向IOC容器中添加组件，如下：

- 组件标注范式(模板)注解 + 包扫描
- 一般在配置类中，使用@Bean注解
- @Import注解，快速导入一个或多个组件
- 使用Spring提供的 FactoryBean
---

### 添加组件
#### 组件标注范式(模板)注解 + 包扫描

```java
@Component(value="myCat")
public class Cat {
}
```

```java
@ComponentScan(basePackages = {"cn.llman"},
        useDefaultFilters = false,
        includeFilters = {
                @ComponentScan.Filter(type = FilterType.ANNOTATION, 
                                      classes = {Controller.class}),
                @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE, 
                                      classes = {Cat.class})
        },
        excludeFilters = {
                @ComponentScan.Filter(type = FilterType.ANNOTATION,
                                      classes = {Service.class}),
                @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE,
                                      classes = {UserService.class})
        }
)
@ComponentScans(value = {
        @ComponentScan(),
        @ComponentScan(),
        @ComponentScan()
})
@Configuration
public class MainConfig {
    
}
```

如上，就是一个典型地通过组件标注范式(模板)注解 和 包扫描的方式向IOC容器中注入组件。

首先，在Cat类上标注了`@Component`注解，表明我们想要向容器中注入该组件，可以使用value来指定注入到容器中组件的id。

然后，在一个配置类上加入`@ComponentScans`或`@ComponentScan`注解，为了演示方便，将这两个注解标注在同一个配置类上。可以看出`@ComponentScans`内部可以包含多个`@ComponentScan`注解，所以此处省略了很多冗余的配置。

在`@ComponentScan`注解中，我们可以配置扫描包的路径，默认情况下，只会扫描该配置类的同级目录和子级目录中的类，可以通过basePackages来扩大扫描的范围。由于我们想自己设定过滤规则，所以需要将默认的过滤规则关闭，即useDefaultFilters置为false，然后通过includeFilters和excludeFilters来添加详尽的过滤规则。

includeFilters和excludeFilters本质上是两个@Filter的数组，所以我们可以以数组的形式添加多个规则。@Filter注解中最为重要的是一个FilterType和一个Class<?>[]，FilterType是一个枚举类，标识我们应当以何种规则进行过滤，默认情况是以注解的类型来过滤，如上面，过滤了@Controller注解。这个枚举类有以下几种：

- FilterType.ANNOTATION：以注解的类型来过滤
- FilterType.ASSIGNABLE_TYPE：以指定类的类型来过滤
- FilterType.ASPECTJ：使用ASPECTJ表达式来过滤
- FilterType.REGEX：使用正则表达式来过滤
- FilterType.CUSTOM：使用自定义规则来过滤

在includeFilters和excludeFilters添加完相应的过滤规则后，Spring就会根据这些规则来过滤Bean。如果满足了includeFilters中的过滤条件将会被添加到IOC容器中；如果满足了excludeFilters中的过滤条件，就无法被添加。它们刚好形成了互补的过滤功能。

所以，最终上面的扫描过滤规则表示的是：添加所有被@Controller注解标注的类，添加Cat类；不添加所有被@Service注解标注的类，不添加UserService类。

@Component注解有几种变体的形式：@Controller，@Service，@Repository。这种组件添加的方式**常用于向IOC容器中添加我们自己的类，所以我们经常在自己的组件上标注这些注解。**

---

#### 使用@Bean注解

一般在配置类中，使用`@Bean`可以向IOC容器中添加组件，这和XML配置中通过<bean>标签添加组件的方式一致。

```java
@Configuration // 标识这是一个Spring的配置类
public class MainConfig {
    // 给容器中注册一个Bean;类型为返回值的类型，id 默认用方法名作为id
    @Bean
    public Person person() {
        return new Person("lma", 21);
    }
}
```

这种组件的添加方式**常用于向IOC容器中注入第三方包中的组件**，所以当我们无法通过修改第三方包的源文件，又想向IOC容器中添加这些组件时，可以使用@Bean注解的方式。

#### 使用@Import注解，快速导入一个或多个组件

一般在配置文件中，我们可以使用`@Import`快速的向容器中注入一个或多个组件。

```java
@Configuration
@Import({Color.class,
        Animal.class,
        MyImportSelector.class,
        MyImportBeanDefinitionRegistrar.class
})// 快速导入组件，id默认是组件的全类名
public class MainConfig {
    
}
```

在上面的配置中，我们通过`@Import`注解向容器中添加了四个组件，前两个组件都是普通的Java类，所以就会向IOC容器中导入这个两个组件，默认的id是组件的全类名。

第三个组件是MyImportSelector类，它实现了Spring的ImportSelector接口：

```java
// 自定义逻辑 返回需要导入的组件
public class MyImportSelector implements ImportSelector {
    /**
     * 返回值就是要导入到容器中的组件的全类名
     * AnnotationMetadata
     */
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[]{"cn.llman.bean.Cat", "cn.llman.bean.Dog"};
    }
}
```

所以，在向容器中导入MyImportSelector组件的时候，也会向容器中注入在这个类中配置了全类名的类。如上的代码中，一次性向IOC容器中导入了3个组件，分别是：MyImportSelector、Cat、Dog。

第四个组件是MyImportBeanDefinitionRegistrar，它实现了Spring的ImportBeanDefinitionRegistrar接口：

```java
public class MyImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    /**
     * AnnotationMetadata
     * BeanDefinitionRegistry: bean 定义的注册类
     * 把所有需要注册到容器中的bean，调用BeanDefinitionRegistry的registerBeanDefinition来手动注册
     */
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, 
                                        BeanDefinitionRegistry registry) {
        boolean existDog = registry.containsBeanDefinition("cn.llman.bean.Dog");
        boolean existCat = registry.containsBeanDefinition("cn.llman.bean.Cat");
        if (existCat && existDog) {
            // 指定bean的信息(bean的类型等..)
            RootBeanDefinition beanDefinition = new RootBeanDefinition(Pig.class);
            System.out.println("Preparing to inject a bean named Pig ...");
            // 注册一个bean，并指定bean的名称
            registry.registerBeanDefinition("Pig", beanDefinition);
        }
    }
}
```

所以在向容器中导入MyImportBeanDefinitionRegistrar组件时，也会执行该类中定义的registerBeanDefinitions方法，这个方法中可以实现很细节性的控制。我们可以通过BeanDefinitionRegistry对象获取IOC容器中Bean的定义信息(BeanDefinition)，可以进行判断后再进行注入，达到了一种条件注入的功能。如上，我们判断只有到Cat和Dog都被在BeanDefinition中时，才去执行注册Pig的BeanDefinition。

什么是BeanDefinition呢？Bean的信息，包含了该Bean的类文件、是否切面增强、是否属性注入、是否支持事务等等，Spring就是根据BeanDefinition中包含的信息来最终实例化该Bean，可以认为BeanDefinition就是Bean的蓝图。

在导入MyImportBeanDefinitionRegistrar的过程中，先导入了MyImportBeanDefinitionRegistrar的实例对象，后再根据判断的条件，可能会再次导入Pig对象。

**这种通过`@Import`向IOC容器中注册Bean的方式在Springboot框架中大量使用，常常和`@Conditional`一起使用，达到条件快速注入Bean**

#### 使用Spring提供的 FactoryBean

在配置文件中我们可以通过@Bean注解的方式向IOC容器中注册一个FactoryBean的实例，在FactoryBean的实例中，可以定义其它Bean的注入信息。

```java
@Configuration 
public class MainConfig {
    @Bean
    public AnimalFactoryBean animalFactoryBean(){
        return new AnimalFactoryBean();
    }
}
```

如上的代码中，我们通过@Bean注解向容器中注入了一个AnimalFactoryBean的实例。这个AnimalFactoryBean实现了Spring的FactoryBean接口。

```java
// 定义一个animal对象的工厂bean
public class AnimalFactoryBean implements FactoryBean<Animal> {
	// 返回一个animal对象，并添加到容器中
    @Override
    public Animal getObject() throws Exception {
        System.out.println("Creating a animal ...");
        return new Animal();
    }
    @Override
    public Class<?> getObjectType() {
        return Animal.class;
    }
    // 该实例是否为单例? true: 是单实例;false: 不是单实例
    @Override
    public boolean isSingleton() {
        return false;
    }
}
```

**这种方式常被Spring用于与其他框架的整合**。默认的情况下，从IOC容器中获取到的是由该工厂Bean通过getObject()方法创建的对象，如果想要获得工厂Bean本身，就需要在id前面加上一个&符号，如：context.getBean("&animalFactoryBean") 。

---

### 条件注入

条件注入是指只有在满足某些条件下，才向IOC容器中注入某些组件。

虽然我们可以在配置类包扫描`@ComponentScan`注解中添加条件，但是，`@ComponentScan`必须配合模式注解(`@Component`)共同使用，有些三方包中的类，我们是没有办法用这种方法来完成。

另外，我们可以使用一个类来实现`ImportBeanDefinitionRegistrar`接口，也能达到条件注入的目的。但是`ImportBeanDefinitionRegistrar`中只能获取到BeanDefinition信息，有些外部的配置，这个类是感知不到的，所以在某些场景下，不尽如人意。

所以，Spring提供了一个专门用了条件注入的注解`@Conditional`：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Conditional {
	Class<? extends Condition>[] value();
}
```

从源码中，我们可以看到，`@Conditional`注解中接收一个数组，数组中元素的类型需要继承自Condition。

如，我们可以根据当前运行的操作系统来条件性的注入一些Bean

```java
// 判断的条件：如果操作系统的名称中包含Windows便通过
public class WindowsCondition implements Condition {
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        // 获取当前运行的环境，JVM OS 等
        Environment environment = context.getEnvironment();
        String osName = environment.getProperty("os.name");
        if (osName.contains("Windows")) {
            // 判断是否为Windows系统
            return true; // 通过条件，可以向IOC容器中添加组件
        }
        return false; // 未通过条件，不能向IOC容器中添加组件
    }
}
```

```java
// 判断的条件：1. 如果BeanDefinition中包含了myCat的信息便通过
//			 2. 如果操作系统的名称中包含了Linux便通过
public class LinuxCondition implements Condition {
    /**
     * ConditionContext 判断条件能否使用的上下文环境
     * AnnotatedTypeMetadata 被注解类型的原信息
     */
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        // 获取IOC容器使用的 BeanFactory
        ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
        // 获取类加载器 ClassLoader
        ClassLoader classLoader = context.getClassLoader();

        // 获取当前运行的环境，JVM OS 等
        Environment environment = context.getEnvironment();
        // 获取bean定义的注册类
        BeanDefinitionRegistry registry = context.getRegistry();

        // 判断容器中bean的注册情况，也可以给容器中注册bean
        boolean existMyCat = registry.containsBeanDefinition("myCat");
        if (existMyCat) {
            return true; 
        }
        
        String osName = environment.getProperty("os.name");
        if (osName.contains("Linux")) {
            // 判断是否为Linux系统
            return true;
        }
        return false;
    }
}
```

从上面的实例代码中，我们可以看到，实现了`Condition`接口后，需要重写一个matches方法，这个方法的第一个参数是ConditionContext实例，通过该实例，我们可以获得一系列的信息：BeanFactory、ClassLoader、BeanDefinitionRegistry、Environment等，拿到这些信息后，基本上IOC的全部信息就在自己的手上了，可以对容器中所有的实例进行详尽的检查和判断，最终返回一个判断后的结果。

当我们需要使用这些实现了`Condition`的类时，就可以使用`@Conditional`注解将这些条件组合起来，这个`@Conditional`注解既可以标注在类上，也可以标注在一个方法上。

```java
@Configuration
@Conditional({WindowsCondition.class}) // 满足该条件，这个配置类所有配置的bean才会生效; 
									// 对类中组件的统一设置
public class MainConfig {  
    @Bean("Bill")
    @Conditional({WindowsCondition.class}) // 满足该条件，这个Bean才会被注入
    public Person bill() {
        return new Person("Bill Gates", 23);
    }
    
    @Bean("Linus")
    @Conditional({LinuxCondition.class})
    public Person linus() {
        return new Person("Linus", 23);
    }
}
```

值得注意的是，在Springboot中大量地使用了`@Conditional`注解，它配合`@Import`注解进行快速地条件注入。

---

### Bean的作用域

通过`@Scope`注解可以在注入Bean的时候调节其作用域

通过查看`@Scope`注解可以看到可配置的作用域有四种情况：

- PROTOTYPE 多实例 

  多实例情况下，在容器启动时不会立即创建该实例对象；每次需要该实例时，才会调用方法创建该实例；并且，每次获取都会创建不同的实例对象

- SINGLETON 单实例 (默认值)

  单实例情况下，在容器启动时就会创建该实例对象；以后每次需要实例时，就直接从容器中获取

- REQUEST 在Web环境下，同一次请求创建一个实例

- SESSION 在Web环境下，同一个会话创建一个实例

```java
@Configuration
public class MainConfig {  
    @Scope(value = "singleton")
    @Lazy
    @Bean // 默认的组件都是单实例的(SCOPE_SINGLETON)
    public Person person() {
        System.out.println("Inject a bean to container ... ");
        return new Person("lma", 23);
    }
}
```

如上的代码，可以通过`@Scope`来修改Bean的作用域，默认情况下都是单实例的。

在单实例的情况下，IOC容器在一启动的时候，就会去创建该实例的Bean。但我们可以通过`@Lazy`注解来告诉IOC容器，在启动时先不要创建该实例，而是等到第一次从容器中获取该实例Bean时，才去创建，并进行初始化，这就是所谓的懒加载。

---

这一小节，系统地了解向IOC容器中添加组件的方式。下一小节，将了解在Spring IOC容器的控制下，一个Bean又经历了怎样的生老病死，也将第一次接触BeanPostProcess，可以译为对象的后置处理器，这是Spring中很重要的一个机制，正是在这个机制下，Spring完成了很多情况下对Bean的增强，我们也可以模仿着Spring的方式，通过自定义的BeanPostProcess去人为地增强一些Bean。

