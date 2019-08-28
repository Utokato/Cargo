## Spring 注解驱动开发(三) 容器管理Bean的生命周期

在Spring中，一个Bean的生命周期都是由Spring来管理，一般经历了以下三个阶段：

- Bean的创建
- Bean的初始化
- Bean的销毁

这里值得注意的是，Bean的创建、构造、实例化，其实表达的是同一个概念，只是称呼有所不同，本质上是IOC容器通过反射机制使用该Bean的类文件，调用构造函数实例化对象的过程。

我们可以自定义Bean的初始化和销毁方法，当容器在执行到Bean相应的生命周期时，就会调用自定义的初始化和销毁方法。

在上节谈到Bean的作用域(Scope)时，我们知道一个单实例(Singleton)Bean，默认在容器启动时就会实例化该对象；而一个多实例(Prototype)Bean，只有在第一次获取对象时才会去创建该对象。

在Spring的使用过程中，我们可以通过不同的方式来自定义Bean的初始化和销毁方法。

### 使用@Bean注解

通过@Bean注解，可以指定 init-method (在对象初始化完成后调用) 和 destroy-method (在容器关闭前调用) 方法。

在bean创建(构造、实例化)完毕后调用(单实例bean在容器创建时就会创建，多实例bean在获取对象是才会创建)指定的init-method方法；在容器关闭前调用destroy-method方法，单实例的bean随着容器的关闭而销毁，多实例的bean，容器只负责bean的创建(构造)，不负责bean的销毁。

```java
public class Car {
    public Car() {
        System.out.println("Car constructor running... ");
    }
    public void init() { // 方法名并不重要，对应设置就可以生效
        System.out.println("Car init...");
    }
    public void destroy() {
        System.out.println("Car destroy...");
    }
}
```

```java
@Configuration
public class MainConfig {
    @Bean(initMethod = "init",destroyMethod = "destroy") // 与该类中定义的方法名相呼应
    public Car car() {
        return new Car();
    }
}
```

```java
public class Test {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext =
                new AnnotationConfigApplicationContext(MainConfig.class);
        Car car = applicationContext.getBean(Car.class);
        // System.out.println(car);
        applicationContext.close();
    }
}
// 控制台上打印的记录
// Car constructor running... 
// Car init...
// Car destroy...
```

对于默认的单实例Bean，会在IOC容器刷新时创建所有的实例，也就会执行这些Bean设置的初始化方法；在调用容器的关闭方法之前，会执行Bean的销毁方法。

### 通过Bean类实现相应的接口

也可以让Bean实现`InitializingBean`接口，在`afterPropertiesSet`方法中定义初始化的逻辑；实现`DisposableBean`接口，在`destroy`方法中定义销毁的逻辑。

```java
public class Autobike implements InitializingBean, DisposableBean {
    public Autobike() {
        System.out.println("Autobike is constructing...");
    }
    @Override
    public void destroy() throws Exception {
        System.out.println("Autobike destroy...");
    }
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("Autobike init...");
    }
}
```

### 使用JSR250规范中的相关注解

通过`@PostConstruct` 注解来标识这个方法在Bean创建完成并属性赋值完成来执行初始化；通过`@PreDestroy`注解来标识这个方法是在容器关闭之前调用。

```java
public class Bicycle {
    public Bicycle() {
        System.out.println("Bicycle is constructing...");
    }
    @PostConstruct // JSR250 中的注解
    public void init() { // 同样的，方法名不重要，重要的是注解标注的位置
        System.out.println("Bicycle init running, after Bicycle construction...");
    }
    @PreDestroy // JSR250 中的注解
    public void destroy() {
        System.out.println("Bicycle method running, before Bicycle destroy...");
    }
}
```

---

### 初识 `BeanPostProcessor`

`BeanPostProcessor` 可以译为Bean的后置处理器，可以在Bean实例化完成，初始化前后做一些控制等。这是Spring中非常重要的一个机制，很多的功能都是依赖于和这个机制。这里需要注意的一点是，Bean的实例化和初始化是不一样的，实例化是Bean的构造，属性赋值等；初始化是指Bean已经创建(构造)了，再做的一些init工作，这个init的方法，就是使用上面三种方法之一定义的初始化。

现在，我们首先简单地了解一下`BeanPostProcessor`接口以及这个接口的一个子接口 `InstantiationAwareBeanPostProcessor`。

在`BeanPostProcessor`接口中只声明了两个方法：

```java
@Nullable // 在初始化方法调用前执行
default Object postProcessBeforeInitialization(Object bean, String beanName) 
    throws BeansException {return bean;}

@Nullable // 在初始化方法调用后执行
default Object postProcessAfterInitialization(Object bean, String beanName) 
    throws BeansException {return bean;}
```

在`InstantiationAwareBeanPostProcessor`接口中声明了三个方法：

```java
@Nullable // 在实例化方法之前调用
default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) 
        throws BeansException {return null;}
// 在实例化方法之后调用
default boolean postProcessAfterInstantiation(Object bean, String beanName) 
    throws BeansException {return true;}

@Nullable // 处理属性值
default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, 
                                             String beanName) 
    throws BeansException {return null;}
```

这两个接口，分别对应了一个Bean的实例化和初始化的阶段，我们可以先看个实例，再来总结其中的执行细节。

首先，我们自己定义一个类，实现`InstantiationAwareBeanPostProcessor`接口，同时也就实现它的父接口，即：

```java
@Component // 将该类注册到IOC容器中
public class MyInstantiationAwareBeanPostProcessor implements 
    InstantiationAwareBeanPostProcessor {
    // 实例化前的方法，此时Bean还没有实例化，所有只能获得该Bean的类文件，无法获得实例对象
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) 
        throws BeansException {
        System.out.println("Notice! A method before instantiation is running, "
                           + "and the bean prepared to instance is: " + beanName);
		return null;
    }
	// 实例化之后的方法，此时Bean已经实例化完毕了，所以可以获得实例对象
    public boolean postProcessAfterInstantiation(Object bean, String beanName) 
        throws BeansException {
        if (bean instanceof Book) {
            Book book = (Book) bean;
            System.out.println("Fortunately! I am the method after instantiation. "
                               + "I can get a book that is: " + book);
        }
        return false;
    }
	// 初始化之前的方法，此时Bean已经实例化，并赋值完毕，所以可以获得实例对象
    public Object postProcessBeforeInitialization(Object bean, String beanName) 
        throws BeansException {
        if (bean instanceof Book) {
            Book book = (Book) bean;
            System.out.println("Notice! I am the method before initialization."
                               +" I can also get a book that is: " + book);
        }
        return null;
    }
	// 实例化之后的方法，此时Bean已经实例化，并经过了初始化，所以可以获得实例对象
    public Object postProcessAfterInitialization(Object bean, String beanName) 
        throws BeansException {
        if (bean instanceof Book) {
            Book book = (Book) bean;
            System.out.println("Ooh! Fortunately! I am the method after initialization,"
                               +" so I can get a book that is: " + book);
            // try to change the info of this book
            // to imitate a enhance for this bean
            book.setName("Pride and Prejudice");
            book.setPage(699);
        }
        return null;
    }
}
```

接着，来看下我们的Book对象，也就是我们要在IOC容器中注入的对象：

```java
// Book，省略了Getter/Setter和toString方法
public class Book {
    private String name;
    private Integer page;
    public Book() { // 无参构造方法
        System.out.println("Bingo! The constructor without args of book is running.");
    }
    public Book(String name, Integer page) { // 有参构造方法
        System.out.println("Bingo! The constructor of book is running.");
        this.name = name;
        this.page = page;
    }
    public void init() { // 定义一个初始化方法
        System.out.println("Try to work with init method.");
        this.setName("War and Peace");
        this.setPage(499);
    }
}
```

然后，我们在一个配置类中，将这个Book对象注入：

```java
@Configuration // 标识这是一个配置类
@ComponentScan({"cn.demo.itest"}) // 确保MyInstantiationAwareBeanPostProcessor能被扫描到
public class BookConfig {
    @Bean(initMethod = "init") // 标识这是一个Bean，并设置初始化方法
    public Book book() {
        Book book = new Book("A Tale of Two Cities", 599); // 通过有参构造方法new一个对象
        return book;
    }
}
```

最后，通过一个测试用例将所有的流程跑起来：

```java
public class BookTest {
    @Test
    public void testOne() {
        AnnotationConfigApplicationContext applicationContext = 
            new AnnotationConfigApplicationContext(BookConfig.class);
        Book book = applicationContext.getBean(Book.class); // 根据类型从IOC容器中获取Bean
        System.out.println("Eureka! I have found a book that detail is: " + book);
        applicationContext.close();
    }
}
```

由于我们在不同的代码中都设置了输出语句，所有程序在执行的过程中，会在控制台打印一些语句，通过分析这些语句，我们就可以判断代码的执行流程。在贴出运行的结果之前，我们可以自己来分析一下书籍名称的变化，根据书籍名称的变化，基本上就可以追踪到代码执行的细节。

首先，在实例化对象之前，是获取不到该对象的，所以只有在实例化(构造函数)对象后，才能获取到该对象，在构造函数中，我们第一次为书籍赋名；然后我们在init方法中，第二次为书籍赋名；最后，在后置处理器的`postProcessAfterInitialization`方法中第三次为书籍赋名。所以，依据这个顺序，书籍的名字变化先后为：

1. A Tale of Two Cities
2. War and Peace
3. Pride and Prejudice

现在，我们来贴出控制台的记录：

```java
Notice! A method before instantiation is running, and the bean prepared to instance is: book
Bingo! The constructor of book is running.
Fortunately! I am the method after instantiation. 
    I can get a book that is: Book[name='A Tale of Two Cities', page=599]
Notice! I am the method before initialization. 
    I can also get a book that is: Book[name='A Tale of Two Cities', page=599]
Try to work with init method.
Ooh! Fortunately! I am the method after initialization, 
	so I can get a book that is: Book[name='War and Peace', page=499]
Eureka! I have found a book that detail is: Book[name='Pride and Prejudice', page=699]
```

对照着控制台的打印记录，基本上就能理解了。这里需要再提出来几个概念，希望不要搞混淆了。第一个是Spring中Bean的实例化(Instantiation)和初始化(Initialization)，第二个是JDK的初始化(Initialization)。

其中，JDK的初始化(Initialization)是指类文件(.class)的正确初始化，这属于虚拟机类加载机制的范畴。在Java中，一个对象在可以被使用之前，必须被正确的初始化，这一点是Java规范规定的。SpringCore中，Bean的实例化(Instantiation)和初始化(Initialization)都是在类文件的初始化之后。SpringBean初始化强调的是经过实例化以后的Bean的初始化，为什么实例化以后的Bean还需要再次初始化(Spring)呢？这是由于Spring认为一个Bean可能需要被增强(支持事务、AOP等)，所以为此留出了一个可以扩展的点。由此，我们其实可以意识到AOP等高级功能的实现，也是基于这个`BeanPostProcessor`机制。从这段的描述中，我们应该能够认识到SpringBeanInitialization和JDKInitialization完全不是一回事。(注：这一段表述有纰漏)

最后，我们对上面的打印记录进行再次描述：

1. 首先是`postProcessBeforeInstantiation`运行，此时bean还没有实例化，所以只能获得bean的名字和类文件。值得注意的是，此时已经能够正常获得相应的class文件，说明JDK的类文件初始化已经结束，所以最早的应该是JVM对类文件的加载；

2. 之后，Spring通过该类的构造器(constructor)实例化对象

3. `postProcessAfterInstantiation`运行，此时实例化对象完毕，所以可以获得bean的实例对象，所以打印出：Book[name='A Tale of Two Cities', page=599]，这也是在构造函数中设置的；

4. `postProcessBeforeInitialization`运行，此时实例化已经结束，但SpringBean的初始化还没有开始，所以获得的Book实例仍为：Book[name='A Tale of Two Cities', page=599]；

5. 然后，指定的初始化方法开始运行，在初始化方法中，我们对书籍的信息进行了修改，从《双城记》修改到了《战争与和平》；

6. `postProcessAfterInitialization`运行，此时SpringBean的实例化和初始化工作都已结束，并且由于初始化中做了一些修改，所以这里获得的bean为：Book[name='War and Peace', page=499]；同时，在这个方法中也可以获得Book对象，我们再次对这个bean进行修改，从《战争与和平》修改到《傲慢与偏见》；

7. 此时，SpringBean经历了上面的流程最终加入了IOC容器中了；

8. 最后，我们在测试方法中，从IOC容器中获取该实例，猜猜我们拿到了什么实例？

   没错，Book[name='Pride and Prejudice', page=699]，就是《傲慢与偏见》了。

---

这一节，我们了解了Bean的生命周期，以及这些生命周期都是如何被正确保障的，也第一次接触了`BeanPostProcessor`，现在需要掌握的就是各种实例化和初始化的时机，后面对这个机制还有更详尽的分析。

