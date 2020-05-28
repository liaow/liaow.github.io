---
title: Spring注解驱动 - 组件装载
categories: spring
---

## Bean生命周期

> [高清图+代码对bean生命周期深度解析](https://mp.weixin.qq.com/s/eYqsvEvwBsnmgDlKVGLLLg)

Spring 只帮我们管理单例模式 Bean 的完整生命周期，对于 prototype 的 bean ，Spring 在创建好交给使用者之后则不会再管理后续的生命周期。

Spring 管理的 bean 的作用域默认是单例，在容器启动的时候创建对象。而作用域是多例的时候，是在每次获取的时候创建对象。

bean 的初始化和销毁方法可以：

1. `@Bean`注解的`init-methdod`和`destroy-method`属性中指定，但是要求传入的是不带参数的方法名。
2. 实现`InitializingBean`和`DisposableBean`接口
3. 使用`@PostConstruct`和`@PreDestroy`注解
4. 实现`BeanPostProcessor`接口（后置处理器）。

<!--more-->

### @Bean 注解属性设置

对要注册的 bean，自定义初始化和销毁方法：

```java
public class Car {
	public Car() {
		System.out.println("car construct...");
	}
	public void init() {
		System.out.println("car init...");
	}
	public void destory() {
		System.out.println("car destory...");
	}
}
```

将自定义的初始化方法和销毁方法名配置到`@Bean`注解属性中：

```java
@Configuration
public class AppConfig6 {
	@Bean(initMethod = "init", destroyMethod = "destory")
	public Car car() {
		return new Car();
	}
}
```

加载配置类，随即关闭容器：

```java
public class TestCode06 {
	@Test
	public void testLifeMethod() {
		AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig6.class);
		System.out.println("容器创建完成");
		applicationContext.close();
	}
}
```

输出结果：

> car construct…
> car init…
> 容器创建完成
> car destory…

当设置 bean 的作用域是多例的时候：

```java
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
@Bean(initMethod = "init", destroyMethod = "destory")
public Car car() {
    return new Car();
}
```

测试结果：

> 容器创建完成

在关闭前获取一次 bean 测试 ：

```java
public class TestCode06 {
	@Test
	public void testLifeMethod() {
		AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig6.class);
		System.out.println("容器创建完成");
		applicationContext.getBean(Car.class);
		applicationContext.close();
	}
}
```

测试结果：

> car construct…
> car init…
> 容器创建完成
> car destory…

因此可以得出：spring 对于多例作用域的 bean，只有在获取 bean 的时候才执行配置的初始化和销毁的方法。

### 实现InitializingBean和DisposableBean接口

要注册的 bean 实现 `InitializingBean`接口和`DisposableBean`接口，自己重写初始化方法和销毁方法：

```java
public class Tank implements InitializingBean, DisposableBean {
	public Tank() {
		System.out.println("tank construct...");
	}
	@Override
	public void destroy() throws Exception {
		System.out.println("tank --> DisposableBean --> destroy");
	}
	@Override
	public void afterPropertiesSet() throws Exception {
		System.out.println("tank --> InitializingBean --> afterPropertiesSet");
	}
}
```

将这个 bean 也配置到配置类中：

```java
@Configuration
public class AppConfig6 {
	// @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
	@Bean(initMethod = "init", destroyMethod = "destory")
	public Car car() {
		return new Car();
	}
	@Bean
	public Tank tank() {
		return new Tank();
	}
}
```

测试：

```java
public class TestCode06 {
	@Test
	public void testLifeMethod() {
		AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig6.class);
		System.out.println("容器创建完成");
		applicationContext.close();
	}
}
```

输出结果：

> car construct…
> car init…
> tank construct…
> tank –> InitializingBean –> afterPropertiesSet
> 容器创建完成
> tank –> DisposableBean –> destroy
> car destory…

### @PostConstruct和@PreDestroy注解

spring 支持对JSR250规范的注解：

`@PostConstruct`：在bean创建完成并且属性赋值完成，来执行初始化方法

`@PreDestroy`：在容器销毁 bean 之前通知我们进行清理工作

```java
public class Dog {
	public Dog() {
		System.out.println("dog construct...");
	}
	@PostConstruct
	public void init() {
		System.out.println("dog init...");
	}
	@PreDestroy
	public void destory() {
		System.out.println("dog destory...");
	}
}
```

测试输出结果：

> dog construct…
> dog init…
> 容器创建完成
> dog destory…

### 实现BeanPostProcessor接口

BeanPostProcessor是Spring IOC容器给我们提供的一个扩展接口。

```java
public interface BeanPostProcessor {
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```

如上接口声明所示，BeanPostProcessor接口有两个回调方法：

当一个BeanPostProcessor的实现类注册到Spring IOC容器后，对于该Spring IOC容器所创建的每个bean实例在初始化方法（如afterPropertiesSet和任意已声明的init方法）调用前，将会调用BeanPostProcessor中的postProcessBeforeInitialization方法。

而在bean实例初始化方法调用完成后，则会调用BeanPostProcessor中的postProcessAfterInitialization方法，整个调用顺序可以简单示意如下：

1. Spring IOC容器实例化Bean
2. 调用BeanPostProcessor的postProcessBeforeInitialization方法
3. 调用bean实例的初始化方法
4. 调用BeanPostProcessor的postProcessAfterInitialization方法

可以看到，Spring容器通过BeanPostProcessor给了我们一个机会对Spring管理的bean进行再加工。比如：我们可以修改bean的属性，可以给bean生成一个动态代理实例等等。一些Spring AOP的底层处理也是通过实现BeanPostProcessor来执行代理包装逻辑的。

> 多例作用域的 bean 不执行该接口的两个方法。

实现`BeanPostProcessor`接口，并重写两个默认的方法：

```java
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {
	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		System.out.println("postProcessBeforeInitialization..." + beanName + " => " + bean);
		return BeanPostProcessor.super.postProcessAfterInitialization(bean, beanName);
	}
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		System.out.println("postProcessBeforeInitialization..." + beanName + " => " + bean);
		return BeanPostProcessor.super.postProcessBeforeInitialization(bean, beanName);
	}
}
```

**注意**：在很多资料中说接口中两个方法不能返回null，如果返回null，那么在后续初始化方法将报空指针异常或者通过getBean()方法获取不到bena实例对象，因为后置处理器从Spring IoC容器中取出bean实例对象没有再次放回IoC容器中。但是本例是使用`spring-context 5.1.8.RELEASE`版本，即使返回 null 也不会报错，也可以正常获取bean。

配置一个 bean 到配置类中 ：

```java
@ComponentScan("com.example.code06")
@Configuration
public class AppConfig6 {
	@Bean
	public Dog dog() {
		return new Dog();
	}
}
```

测试：

```java
public class TestCode06 {	
	@Test
	public void testLifeMethod() {
		AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig6.class);
		System.out.println("容器创建完成");
		applicationContext.close();
	}
}
```

输出结果：

> dog construct…
> postProcessBeforeInitialization…dog => com.example.code06.Dog@a4102b8
> dog init…
> postProcessBeforeInitialization…dog => com.example.code06.Dog@a4102b8
> 容器创建完成
> dog destory…

从输出结果可以看到：在 bean 构造完成，在执行初始化方法的前后可以自定义一些逻辑，这极大的增强了spring 的扩展性

`org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory`类中的源码体现了上述接口的执行逻辑：

```java
// 给bean进行属性赋值
populateBean(beanName, bd, bw);
// 初始化 bean 操作
initializeBean(beanName, existingBean, bd){
    // 初始化之前回调
    wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    // 调用自定义的初始化方法
    invokeInitMethods(beanName, wrappedBean, mbd);
    // 初始化之后回调
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
}
```

## 属性赋值

在Spring框架中，属性的注入我们有多种方式，我们可以通过构造方法注入，可以通过set方法注入，也可以通过p名称空间注入，方式多种多样，对于复杂的数据类型比如对象、数组、List集合、map集合、Properties等，我们也都有相应的注入方式。其中比较常用的是set注入的方式，下面来看看spring的Set注入的方式：

### 使用@Value赋值

使用`@Value`注解赋值 bean 属性

```java
@Data
public class Person {
	// 基本数值
	@Value("example")
	private String name;
	
	// 可以写SpEL； #{}
	@Value("#{20+1}")
	private String age;
}
```

编写测试：

```java
public class TestCode07 {
	AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig7.class);

	@Test
	public void testValue() {
		Person person = applicationContext.getBean(Person.class);
		System.out.println(person);
		applicationContext.close();
	}
}
```

输出结果：

> Person(name=example, age=21)

### 从配置文件中读取

xml 版本：

```xml
<context:property-placeholder location="classpath:person.properties"/>
```

使用`@PropertySource`读取外部配置文件中的k/v保存到运行的环境变量中，加载完外部的配置文件以后使用`${}`取出配置文件的值：

创建配置文件application.properties，并存一份配置：

```properties
person.nickName=exam
```

配置类中增加`@PropertySource`注解，引入外部配置文件：

```java
@PropertySource(value = {"classpath:/application.properties"})
@Configuration
public class AppConfig7 {

	@Bean
	public Person person() {
		return new Person();
	}
}
```

使用`@Value`注解的`${}`表达式取出容器中的配置信息：

```java
@Data
public class Person {
	// 基本数值
	@Value("example")
	private String name;
	
	// 可以写SpEL； #{}
	@Value("#{20+1}")
	private String age;
	
	// 可以写${} 取出配置文件(properties)中的值（在运行环境变量里面的值）
	@Value("${person.nickName}")
	private String nickName;
}
```

另外，在容器对象中也可以直接获取：

```java
public class TestCode07 {
	AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig7.class);

	@Test
	public void testValue() {
		Person person = applicationContext.getBean(Person.class);
		System.out.println(person);

		String property = applicationContext.getEnvironment().getProperty("person.nickName");
		System.out.println(property);
		applicationContext.close();
	}
}
```

输出结果：

> Person(name=example, age=21, nickName=exam)
> exam

## 自动装配

Spring利用依赖注入和DI完成对IOC容器中各个组件的依赖关系赋值。自动装配的优点有：自动装配可以大大地减少属性和构造器参数的指派。自动装配也可以在解析对象时更新配置。自动装配的方式有很多，其中包含Spring的注解以及java自带的注解。

### @Autowired

在xml 方式的时候：

```xml
<!-- 该 BeanPostProcessor 将自动对标注 @Autowired 的 Bean 进行注入 -->     
<bean class="org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor"/>
```

Spring 提供了`@Autowired`注解可以对成员变量、方法和构造函数进行标注，来完成自动装配的工作。无需再通过传统的在 bean 的xml文件中进行bean的注入配置。

@Autowired是根据类型进行标注的，如需要按照名称进行装配，则需要配合`@Qualifier`使用：

1. 默认优先按照 bean 的类型去容器中找对应的组件
2. 若有多个相同类型的组件，再将属性名称作为组件的 id 去容器中查找
3. 使用`@Qualifier("bookDao")`来指定需要装配的组件id而不是根据属性名
4. 使用`@Autowired`注解的时候，如果容器中没有这个类型的 bean 就会报异常，可以显式指定`@Autowired(required=false)`避免报错
5. `@Primary`让Spring进行自动装配时，在没有明确用`@Qualifier`指定的情况下默认使用优先首选的 bean

创建 service 接口，并自定义两个实现类：

```java
public interface UserService {
}

@Service
public class UserServiceImpl1 implements UserService {
}

@Service
public class UserServiceImpl2 implements UserService {
}
```

自定义 cotroller 应用 service 接口：

```java
@ToString
@Controller
public class UserController {
	@Autowired
	private UserService userService;
}
```

配置包扫描：

```java
@ComponentScan(basePackages = {"com.example.code08"})
@Configuration
public class AppConfig8 {}
```

编写测试：

```java
public class TestCode08 {
	AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig8.class);
	@Test
	public void testAutowired() {
		System.out.println(applicationContext.getBean(UserController.class));
	}
}
```

运行会报异常：

```java
警告: Exception encountered during context initialization - cancelling refresh attempt: org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'userController': Unsatisfied dependency expressed through field 'userService'; nested exception is org.springframework.beans.factory.NoUniqueBeanDefinitionException: No qualifying bean of type 'com.example.code08.service.UserService' available: expected single matching bean but found 2: userServiceImpl1,userServiceImpl2
```

表示容器中有多个 service 类型的 bean ，`@Autowired`注解的属性名是：userService，找不到 id 为这个属性名的 bean，所以抛出异常。

因此可以修改属性名找到对应的 bean：

```java
@ToString
@Controller
public class UserController {
	@Autowired
	private UserService userServiceImpl1;
}
```

输出结果：

> UserController(userServiceImpl1=com.example.code08.service.UserServiceImpl1@501edcf1)

这种方式不是很优雅，属性名改了，所有用到这个属性变量名的地方都需要改，因此可以使用`@Qualifier`注解，显式指定容器中的 bean：

```java
@ToString
@Controller
public class UserController {
	@Qualifier("userServiceImpl2")
	@Autowired
	private UserService userService;
}
```

输出结果：

> UserController(userService=com.example.code08.service.UserServiceImpl2@78b729e6)

还有一种办法就是，对某个组件使用`@Primary`注解，这个注解表示，如果存在相同类型的 bean，就以这个为主依赖。

```java
@Primary
@Service
public class UserServiceImpl2 implements UserService {
}

@ToString
@Controller
public class UserController {
	@Autowired
	private UserService userServiceImpl1;
}
```

输出结果：

> UserController(userServiceImpl1=com.example.code08.service.UserServiceImpl2@3c9d0b9d)

### @Resource和@Inject

Spring还支持使用`@Resource`（JSR250）和`@Inject`（JSR330）

**@Resource**

可以和`@Autowired`一样实现自动的装配，默认是按照组件的名称来进行装配，不支持`@Primary` 也没有支持和`@Autowired(required = false)`一样的功能。

**@Inject**

需要导入javax.inject的包，和`@Autowired`的功能一样，不支持和`@Autowired(required = false)`一样的功能。

```xml
<dependency>
    <groupId>javax.inject</groupId>
    <artifactId>javax.inject</artifactId>
    <version>1</version>
</dependency>
```

### @Autowired可标注的地方

我们从`@Autowired`这个注解点进去看一下源码，我们可以发现这个注解可以标注的位置有：构造器，参数，方法，属性；都是从容器中来获取参数组件的值：

#### 标注在方法的参数上

`@Autowired`可以标注在方法的参数上，如果容器里只有一种类型的bean，那么在`@Bean`注解这个方法的时候，可以省略不写：

```java
@ComponentScan(basePackages = {"com.example.code08"})
@Configuration
public class AppConfig8 {
	
	@Bean
	public User user(UserController userController) {
		System.out.println(userController.getClass());
		return new User();
	}
    
  // 以上等价于以下
  @Bean
	public User user(@Autowired UserController userController) {
		System.out.println(userController.getClass());
		return new User();
	}
}
```

#### 标注在方法上

`@Autowired`可以注解在属性上，也可以注解在 setter 方法上，这种很不常用：

```java
@Autowired
public void setUserService(UserService userService){
    this.userService = userService;
}
```

#### 标注在构造函数上

`@Autowired`可以标注在构造函数上，需要注意的是，这种也需要注意参数名称和容器中同类型bean 的问题，解决办法还是在方法参数前面注解`@Qualifier`显式指定bean：

```java
@Autowired
public UserController(UserService userServiceImpl2) {
    this.userService = userServiceImpl2;
}
```

输出结果：

> UserController(userService=com.example.code08.service.UserServiceImpl2@6b26e945)

显式指定：

```java
@Autowired
public UserController(@Qualifier("userServiceImpl1") UserService userService) {
    this.userService = userService;
}
```

输出结果：

> UserController(userService=com.example.code08.service.UserServiceImpl1@54c562f7)

综上，这种配置也是常用。

## 使用Spring底层组件

要想使用Spring底层组件，如`ApplicationContext`、`BeanFactory`等。假设有个 Food 组件需要使用`ApplicationContext`对象，那么可以实现`ApplicationContextAware`接口，重写`setApplicationContext()`方法：

```java
@Data
@Component
public class Food implements ApplicationContextAware, BeanNameAware, EmbeddedValueResolverAware {
	private ApplicationContext applicationContext;

	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
		this.applicationContext = applicationContext;
	}

	@Override
	public void setEmbeddedValueResolver(StringValueResolver resolver) {
		String resolveStringValue = resolver.resolveStringValue("current os name = ${os.name}");
		System.out.println(resolveStringValue);
	}

	@Override
	public void setBeanName(String name) {
		System.out.println("current bean name = " + name);
	}
}
```

Spring 提供了很多`Aware`接口的子类接口，并且提供了回调方法，用于我们自定义组件实现这些接口，重写这些回调方法就能拿到 Spring 底层的对象了。`ApplicationContextAware`接口获取IOC容器，`BeanNameAware`接口获取当前 bean 在容器中的 name，`EmbeddedValueResolverAware`解析 Spring 中支持的表达式。

## xxxAware接口回调原理

所有xxxAware都会被ApplicationContextAwareProcessor实现类调用。看`org.springframework.context.support.ApplicationContextAwareProcessor`源码：

```java
@Override
@Nullable
public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
    AccessControlContext acc = null;

    if (System.getSecurityManager() != null &&
        (bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
         bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
         bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
        acc = this.applicationContext.getBeanFactory().getAccessControlContext();
    }

    if (acc != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareInterfaces(bean);
            return null;
        }, acc);
    }
    else {
        invokeAwareInterfaces(bean);
    }

    return bean;
}

private void invokeAwareInterfaces(Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof EnvironmentAware) {
            ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
        }
        if (bean instanceof EmbeddedValueResolverAware) {
            ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
        }
        if (bean instanceof ResourceLoaderAware) {
            ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
        }
        if (bean instanceof ApplicationEventPublisherAware) {
            ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
        }
        if (bean instanceof MessageSourceAware) {
            ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
        }
        if (bean instanceof ApplicationContextAware) {
            ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
        }
    }
}
```

在`invokeAwareInterfaces()`方法中，依次找对应的接口，并调用对应的接口方法，完成传参操作。

## @Profile根据环境装载

Spring为我们提供的可以根据当前环境，动态的激活和切换一系列组件的功能，实际开发中分为：开发环境、测试环境和生产环境。

`@Profile`注解可以指定组件在哪个环境下才能被注册到容器中，加了环境标识的bean，只有这个环境被激活的时候才能注册到容器中，默认是default。

环境激活的两种方法：

- 使用命令行动态参数：在JVM参数配置`-Dspring.profiles.active=test`
- 代码的方式激活某种环境

自定义一个数据库源：

```java
@Data
public class MyDataSource {
	private String jdbcUrl;
}
```

配置多个数据源：

```java
@Configuration
public class AppConfig9 {
	@Profile("test")
	@Bean("testDataSource")
	public MyDataSource testDataSource() {
		MyDataSource dataSource = new MyDataSource();
		dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/test");
		return dataSource; 
	}
	
	@Profile("prod")
	@Bean("prodDataSource")
	public MyDataSource prodDataSource() {
		MyDataSource dataSource = new MyDataSource();
		dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/prod");
		return dataSource; 
	}
	
	@Profile("dev")
	@Bean("devDataSource")
	public MyDataSource devDataSource() {
		MyDataSource dataSource = new MyDataSource();
		dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/dev");
		return dataSource; 
	}
	
	@Bean("dataSource")
	public MyDataSource dataSource() {
		MyDataSource dataSource = new MyDataSource();
		dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/default");
		return dataSource; 
	}
}
```

编写测试类：

```java
public class TestCode09 {
	@Test
	public void testProfile() {
		AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig9.class);
		MyDataSource dataSource = applicationContext.getBean(MyDataSource.class);
		System.out.println(dataSource);
	}
}
```

输出结果：

> MyDataSource(jdbcUrl=jdbc:mysql://localhost:3306/default)

可以使用JVM命令行参数指定运行环境，也可以在容器初始化的时候代码指定：

```java
@Test
public void testProfileCode() {
    //1、创建一个applicationContext
    AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext();
    //2、设置需要激活的环境
    applicationContext.getEnvironment().setActiveProfiles("dev");
    //3、注册主配置类
    applicationContext.register(AppConfig9.class);
    //4、启动刷新容器
    applicationContext.refresh();

    String[] beanNames = applicationContext.getBeanNamesForType(MyDataSource.class);
    for (String beanName : beanNames) {
        System.out.println(beanName);
    }
}
```

输出结果：

> devDataSource
> dataSource

## 总结

- Bean 加载顺序依次为：

  1. 实例化;
  2. 设置属性值;
  3. 如果实现了BeanNameAware接口,调用setBeanName设置Bean的ID或者Name;
  4. 如果实现BeanFactoryAware接口,调用setBeanFactory 设置BeanFactory;
  5. 如果实现ApplicationContextAware,调用setApplicationContext设置ApplicationContext
  6. 调用BeanPostProcessor的预先初始化方法;
  7. 调用InitializingBean的afterPropertiesSet()方法;
  8. 调用定制init-method方法；
  9. 调用BeanPostProcessor的后初始化方法;

- Bean 对象的初始化和销毁方法：

  1）使用 @Bean 注解的属性配置

  2）实现InitializingBean和DisposableBean接口

  3）使用@PostConstruct和@PreDestroy注解

  4）实现BeanPostProcessor接口，能够在bean的属性装配成功之后，在初始化方法执行的前后增加逻辑

- 属性赋值可以使用 @Value 注解，其属性可以配置 “${}” 占位符，配合 @PropertySource 注解引入外部配置文件，然后就可以动态为属性赋值。

- 组件的自动装配：

  1）使用 @Atuowired 注解进行自动装配，当容器中存在多个相同类型的 bean 的时候，可以使用 @Qualifier 注解进行显式指定。

  2）使用@Resource或@Inject

- 想要使用 spring 底层的组件，如 spring ioc 容器可以实现 ApplicationContextAware 接口。当然也可以直接在让spring 通过构造函数传入。

- 由于在开发中会根据不同的软件环境来装载不同的相同类型的不同组件，如根据软件环境装配数据源，就就可以使用 @Profile 注解。
