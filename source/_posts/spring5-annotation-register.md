---
title: Spring注解驱动 - 组件注册
---

# 1. 注解驱动开发

传统Spring、SpringMVC框架开发的时候，经常需要配置繁琐的xml，Spring4开始已经支持注解的java配置方式来代替xml配置了，在Springboot和SpringCloud使用到大量的注解来进行配置，熟练掌握了Spring的注解驱动，就能在使用Springboot和SpringCloud框架的时候轻松自如。

## 1.1 环境准备

这里使用`spring-context`来演示工程demo，pom 依赖如下：
<!--more-->
```xml
<dependencies>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.8</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>5.1.8.RELEASE</version>
    </dependency>
    <dependency>
        <groupId>javax.inject</groupId>
        <artifactId>javax.inject</artifactId>
        <version>1</version>
    </dependency>
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
        <version>4.12</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

# 2. 组件注册

创建一个`Person`类：

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Person {
	private String name;
	private Integer age;	
}
```

## 2.1 xml方式注册

创建一个 xml 配置文件，用于注册这个 Person 对象：

```xml
<beans xmlns声明省略...>
	<bean id="person" class="com.example.code01.Person">
		<property name="age" value="20"></property>
		<property name="name" value="example"></property>
	</bean>
</beans>
```

使用`ClassPathXmlApplicationContext`加载这个配置文件：

```java
@Test
public void testLoadXml() {
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("beans.xml");
    Person person = (Person) applicationContext.getBean("person");
    System.out.println(person);
}
```

输出结果：

> Person(name=example, age=20)

## 2.2 注解方式注册

使用注解的方式进行加载 bean，就需要编写一个配置类，等同于 xml 配置文件：

- `@Bean`：相当于xml配置文件中的`<bean>`标签，告诉容器注册一个bean
- 之前xml文件中`<bean>`标签有bean的class类型，那么现在注解方式的类型当然也就是返回值的类型
- 之前xml文件中`<bean>`标签有bean的id，现在注解的方式默认用的是方法名来作为bean的id，也就是说当前注解的方法名是`person()`，那么id 就是`person`，如果方法名是`person01()`，那么id 就是`person01`

```java
@Configuration
public class AppConfig {
	@Bean
    public Person person() {
        return new Person("example", 20);
    }
}
```

如果通过`@Bean`注解的`value`属性显式指定 bean 在 IOC 容器的id，那么就会以这个指定的id 为准注入容器中：

```java
@Configuration
public class AppConfig {
	@Bean("person")
    public Person person01() {
        return new Person("example2", 22);
    }
}
```

使用`AnnotationConfigApplicationContext`加载这个配置类：

```java
@Test
public void testLoadAnnotation() {
    ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
    Person person = applicationContext.getBean(Person.class);
    System.out.println(person);
    // 遍历所有 Person 类型的 bean 的id
    printBeanName(applicationContext, Person.class);
}

private void printBeanName(ApplicationContext applicationContext, Class clazz) {
    String[] beanNames = applicationContext.getBeanDefinitionNames();
    for (String beanName : beanNames) {
        System.out.println(beanName);
    }
}
```

输出结果：

> Person(name=example2, age=22)
> org.springframework.context.annotation.internalConfigurationAnnotationProcessor
> org.springframework.context.annotation.internalAutowiredAnnotationProcessor
> org.springframework.context.annotation.internalCommonAnnotationProcessor
> org.springframework.context.event.internalEventListenerProcessor
> org.springframework.context.event.internalEventListenerFactory
> appConfig
> person

从输出结果可以看到：配置类也作为了 bean 注入容器中，当使用@Bean 注解显示指定id 的时候，注册到容器中就是这个指定的id。

## 2.3 组件自动扫描组件

在xml文件配置的方式，我们可以这样来进行配置：

```xml
<!-- 包扫描、只要标注了@Controller、@Service、@Repository，@Component -->
<context:component-scan base-package="com.example.code01"/>
```

以前是在xml配置文件里面写包扫描，现在我们可以在配置类里面写包扫描：

```java
@ComponentScan(basePackages = {"com.example.code01"})
@Configuration
public class AppConfig2 {}
```

`@ComponentScan`注解会扫描当前包及其子包的所有带`@Component`、`@Controller`、`@Service`、`@Repository`注解的组件。

```java
@Test
public void testComponentScan() {
    ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig2.class);
    String[] definitionNames = applicationContext.getBeanDefinitionNames();
    for (String name : definitionNames) {
        System.out.println(name);
    }
}
```

输出结果：

> appConfig2
> appConfig
> book
> bookController
> bookDao
> bookService
> person

在 `@ComponentScan` 这个注解上，也是可以指定要排除哪些包或者是只包含哪些包来进行管理：里面传是一个Filter[]数组：

```java
// @ComponentScan 源码
Filter[] includeFilters() default {};
Filter[] excludeFilters() default {};
```

xml 版本：

```xml
<!--包扫描，只要注解了@Component，@Controller等会被扫描-->
<context:component-scan base-package="com.example.code01" use-default-filters="false" >
    <!--排除某个注解,除了Controller其他类都会被扫描-->
    <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
    <!--只包含某个注解-->
    <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```

注解版：

```java
@ComponentScan(basePackages = {"com.example.code01"},
					excludeFilters = { 
							@Filter(type = FilterType.ANNOTATION, classes = {Controller.class, Service.class})
					})
@Configuration
public class AppConfig2 {}
```

其中Filter的type的类型有:

1. `FilterType.ANNOTATION` 按照注解
2. `FilterType.ASSIGNABLE_TYPE` 按照类型 FilterType.REGEX 按照正则
3. `FilterType.ASPECTJ` 按照ASPECJ表达式规则
4. `FilterType.CUSTOM` 使用自定义规则

> excludeFilters = Filter[] 指定在扫描的时候按照什么规则来排除组件
>
> includeFilters = Filter[] 指定在扫描的时候，只需要包含组件

其中自定义规则类型过滤器需要实现TypeFilter接口：

```java
@Component
public class MyTypeFilter implements TypeFilter {
	/**
	 * metadataReader:读取到的当前正在扫描的类的信息
	 * metadataReaderFactory:可以获取到其他任何类信息的
	 */
	@Override
	public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory)
			throws IOException {
		//获取当前类注解的信息
		AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
		//获取当前正在扫描的类的类信息
		ClassMetadata classMetadata = metadataReader.getClassMetadata();
		//获取当前类资源（类的路径）
		Resource resource = metadataReader.getResource();
		String className = classMetadata.getClassName();
		if(className.contains("Dao")){
			System.out.println("--->"+className);
			return true;
		}
		return false;
	}
}
```

`FilterType.CUSTOM`过滤器过滤：

```java
@ComponentScan(basePackages = {"com.example.code02"},
					includeFilters = {
							@Filter( type=FilterType.CUSTOM, classes= { MyTypeFilter.class })
					}, useDefaultFilters = false)
@Configuration
public class AppConfig2 {}
```

输出结果：

> —>com.example.code01.dao.BookDao
> org.springframework.context.annotation.internalConfigurationAnnotationProcessor
> org.springframework.context.annotation.internalAutowiredAnnotationProcessor
> org.springframework.context.annotation.internalCommonAnnotationProcessor
> org.springframework.context.event.internalEventListenerProcessor
> org.springframework.context.event.internalEventListenerFactory
> appConfig2
> bookDao

`FilterType.ANNOTATION`注解过滤：

```java
@ComponentScan(basePackages = {"com.example.code02"},
					includeFilters = {
							@Filter( type=FilterType.ANNOTATION, classes= { Service.class })
					}, useDefaultFilters = false)
@Configuration
public class AppConfig2 {}
```

输出结果：

> org.springframework.context.annotation.internalConfigurationAnnotationProcessor
> org.springframework.context.annotation.internalAutowiredAnnotationProcessor
> org.springframework.context.annotation.internalCommonAnnotationProcessor
> org.springframework.context.event.internalEventListenerProcessor
> org.springframework.context.event.internalEventListenerFactory
> appConfig2
> bookService

我们还可以用 `@ComponentScans`来定义多个扫描规则：里面是`@ComponentScan`规则的数组（但是这样写的话，就必须要 java8 及以上的支持）：

```java
@ComponentScans({@ComponentScan("com.example.code01"),
		@ComponentScan("com.example.code03")
})
@ComponentScan(basePackages = {"com.example.code02"},
					includeFilters = {
							@Filter( type=FilterType.ANNOTATION, classes= { Service.class })
					}, useDefaultFilters = false)
@Configuration
public class AppConfig2 {}
```

## 2.4 bean 的作用域

spring 默认将所有的bean 以单例的形式注入，在容器初始化之前就会创建这个实例，直到容器关闭，会一直存在这个唯一的对象。

> 注意：当作用域为单例的时候，IOC容器在启动的时候，就会将容器中所有作用域为单例的bean的实例给创建出来；以后的每次获取，就直接从IOC容器中来获取，相当于是从map.get()的一个过程；也就是 spring 在任何人没有获取 bean 的时候就缓存一份实例。

配置类：

```java
@ComponentScan(basePackages= {"com.example.code03"})
@Configuration
public class AppConfig3 {}
```

创建一个 bean 对象：

```java
@Component
public class Student {}
```

多次从容器获取 bean ：

```java
public class TestCode03 {
	ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig3.class);
	@Test
	public void testScopes() {
		Student student1 = applicationContext.getBean(Student.class);
		Student student2 = applicationContext.getBean(Student.class);
		System.out.println(student1);
		System.out.println(student2);
        System.out.println(student1 == student2);
	}
}
```

输出结果：

> com.example.code03.Student@b62fe6d
> com.example.code03.Student@b62fe6d
> true

我们可以用`@Scope`这个注解来指定作用域的范围：这个就相当于在xml文件中配置的``标签里面指定`prototype`属性：

```java
/**
 	* Specifies the name of the scope to use for the annotated component/bean.
	 * <p>Defaults to an empty string ({@code ""}) which implies
	 * {@link ConfigurableBeanFactory#SCOPE_SINGLETON SCOPE_SINGLETON}.
	 * @since 4.2
	 * @see ConfigurableBeanFactory#SCOPE_PROTOTYPE
	 * @see ConfigurableBeanFactory#SCOPE_SINGLETON
	 * @see org.springframework.web.context.WebApplicationContext#SCOPE_REQUEST
	 * @see org.springframework.web.context.WebApplicationContext#SCOPE_SESSION
	 * @see #value
	 */
@AliasFor("value")
String scopeName() default "";
```

从源码的注释上，我们可以知道scopeName可以取下面这些值：

- `ConfigurableBeanFactory#SCOPE_PROTOTYPE`

  单实例的（默认值）：ioc容器启动会调用方法创建对象放到ioc容器中。以后每次获取就是直接从容器中拿。

- `ConfigurableBeanFactory#SCOPE_SINGLETON`

  多实例的：ioc容器启动并不会去调用方法创建对象放在容器中。每次获取的时候才会调用方法创建对象。

- `org.springframework.web.context.WebApplicationContext#SCOPE_REQUEST`

  同一次请求创建一个实例

- `org.springframework.web.context.WebApplicationContext#SCOPE_SESSION`

  同一个session创建一个实例

配置原型模式（ConfigurableBeanFactory#SCOPE_PROTOTYPE）：

```java
@ComponentScan(basePackages= {"com.example.code03"})
@Configuration
public class AppConfig3 {
	@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
	@Bean
	public Student student() {
		System.out.println("drop student into Spring IOC ...");
        return new Student();
	}
}
```

测试输出结果：

> drop student into Spring IOC …
> drop student into Spring IOC …
> com.example.code03.Student@78047b92
> com.example.code03.Student@8909f18
> false

我们可以发现，我们用`getBean()`方法获取几次，就创建几次bean的实例；也就是说当bean是作用域为多例的时候，IOC容器启动的时候，就不会去创建bean的实例的，而是当我们调用`getBean()`获取的时候去创建bean的实例；而且每次调用的时候，都会创建bean的实例；

## 2.5 懒加载

`@Lazy`注解可以在容器启动不创建对象。第一次使用(获取)Bean创建对象，并初始化：

```java
@Lazy
@Bean
public Student student() {
    System.out.println("drop student into Spring IOC ...");
    return new Student();
}
```

## 2.6 @Conditional 按照条件注册bean

`@Conditional`：按照一定的条件进行判断，满足条件给容器中注册bean

举例：要求容器根据操作系统注入不同的bean，如果系统是 windows，给容器中注册(“bill”)，如果是linux系统，给容器中注册(“linus”)。

实现`Condition`接口，并重写匹配条件逻辑：

windows 条件：

```java
public class WindowsCondition implements Condition {
	@Override
	public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
		// 获取当前容器的环境信息
		Environment environment = context.getEnvironment();
		// 获取系统的名称
		String osName = environment.getProperty("os.name");
		if(osName.contains("Windows")){
			return true;
		}
		return false;
	}
}
```

linux 条件：

```java
public class LinuxCondition implements Condition {
	@Override
	public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
		// 获取当前容器的环境信息
		Environment environment = context.getEnvironment();
		// 获取系统的名称
		String osName = environment.getProperty("os.name");
		System.out.println("osName --> " + osName);
		if(osName.contains("Linux")){
			return true;
		}
		return false;
	}
}
```

`@Conditional`按照一定条件进行判断，满足条件容器中注册bean，若放在类中，整个配置类中的bean满足条件才会被加载到容器中：

```java
@Configuration
public class AppConfig4 {
	@Conditional(WindowsCondition.class)
	@Bean("bill")
	public Boss boss1(){
		return new Boss("Bill Gates", 62);
	}
	@Conditional(LinuxCondition.class)
	@Bean("linus")
	public Boss boss2(){
		return new Boss("linus", 48);
	}
}
```

测试一下注入容器的 id 是谁：

运行测试会发现，spring 只注入了 id 为 linus 的 bean 对象。

```java
public class TestCode04 {
	ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig4.class);
	@Test
	public void testCondition() {
		String[] definitionNames = applicationContext.getBeanDefinitionNames();
        for (String name : definitionNames) {
            System.out.println(name);
        }
	}
}
```

在 windows 系统运行，可以看到只输出了 id 为 bill 的 bean，可以在 JVM 运行参数中手动设置系统环境参数：`-Dos.name=Linux`，再次运行，则输出了 id 为 linus 的 bean

## 2.7 @Import 导入组件

### 2.7.1 导入 bean 的类对象

`@Import(bean的类对象)`可以快速导入一个 bean ，而不需要使用`@Bean`注解一个方法的形式：只要将要注册bean 的类对象传进去，容器就能自动注册这个组：

```java
@Import(Color.class)
@Configuration
public class AppConfig5 {}
```

测试是否导入成功：

```java
public class TestCode05 {
	ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig5.class);
	@Test
	public void testImport() {
		String[] definitionNames = applicationContext.getBeanDefinitionNames();
        for (String name : definitionNames) {
            System.out.println(name);
        }
	}
}
```

输出结果：

> appConfig5
> com.example.code05.Color

从输出结果可以看出：`@Import(bean的类对象)`注册的 id 默认是全类名。

### 2.7.2 导入 ImportSelector 的实现类类对象

实现`ImportSelector`接口，重写`selectImports()`方法，返回要注册的 bean 对象的全类名：

```java
public class MyImportSelector implements ImportSelector {
	@Override
	public String[] selectImports(AnnotationMetadata importingClassMetadata) {
		return new String[]{"com.example.code05.Blue"};
	}
}
```

导入 实现类

```java
@Import({Color.class, MyImportSelector.class})
@Configuration
public class AppConfig5 {}
```

### 2.7.3 手动注册 bean

实现`ImportBeanDefinitionRegistrar`接口，重写注册 bean 的注册方法，将所有要注册的 bean 手动注册 进容器：

```java
/**
 * AnnotationMetadata:当前类的注解信息
 * BeanDefinitionRegistry:BeanDefinition注册类
 */
@Override
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    String[] beanNames = registry.getBeanDefinitionNames();
    for (String beanName : beanNames) {
        System.out.println("registerBeanDefinitions --> " + beanName);
    }
    boolean blue = registry.containsBeanDefinition("com.example.code05.Blue");
    boolean color = registry.containsBeanDefinition("com.example.code05.Color");
    if(color && blue) {
        // 手动创建一个 BeanDefinition 并注册到容器中
        RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(RainBow.class);
        registry.registerBeanDefinition("rainBow", rootBeanDefinition);
    }
}
```

将这个实现类导入到容器中：

```java
@Import({Color.class, MyImportSelector.class, MyImportBeanDefinitionRegistrar.class})
@Configuration
public class AppConfig5 {}
```

输出结果：

> appConfig5
> com.example.code05.Color
> com.example.code05.Blue
> rainBow

**注意**：当所有组件都注册成功之后，才执行这个实现类的注册方法。

## 2.8 使用FactoryBean注册组件

首先创建一个类实现`FactoryBean`接口，其中T是要注册的Bean的类型：

```java
public class ColorFactoryBean implements FactoryBean<Color> {
	@Override
	public Color getObject() throws Exception {
		System.out.println("getObject --> new Color()");
		return new Color();
	}
	@Override
	public Class<?> getObjectType() {
		return Color.class;
	}
	/**
	 * 	接口的默认方法，默认返回true，表示单例模式
	 * 	返回 false 表示每次获取 bean 的时候就会调用getObject()方法
	 */
	@Override
	public boolean isSingleton() {
		return false;
	}
}
```

将工厂类注册到配置类中：

```java
@Configuration
public class AppConfig5 {
	@Bean
	public ColorFactoryBean colorFactoryBean() {
		return new ColorFactoryBean();
	}
}
```

编写测试类：

```java
@Test
public void testFactoryBean() {
    Color colorFactoryBean1 = (Color)applicationContext.getBean("colorFactoryBean");
    Color colorFactoryBean2 = (Color)applicationContext.getBean("colorFactoryBean");
    System.out.println("bean class type --> " + colorFactoryBean1.getClass());
    System.out.println(colorFactoryBean1 == colorFactoryBean2);
    // 在id前面增加 &，获取工厂对象本身
    Object bean = applicationContext.getBean("&colorFactoryBean");
    System.out.println("bean class type --> " + bean.getClass());
}
```

输出结果：

> getObject –> new Color()
> getObject –> new Color()
> bean class type –> class com.example.code05.Color
> false
> bean class type –> class com.example.code05.ColorFactoryBean

# 3. 总结

xml 配置文件在 spring 注解开发中已经被替换成了Java 配置类，要成为具有 xml 功能的配置类，就需要在类名上增加`@Configuration`注解。

bean 组件注册的方式有以下：

1. 使用`@Bean`注解，编写创建 bean 并返回这个bean 的共有方法，在这个方法上加注解，默认会将方法名作为bean 的 id。

容器默认装载的 bean 都是单例的，如果需要多例的 bean ，就需要添加`@Scope`注解。

2. 实现`Condition`接口，并自定义条件规则，使用注解`@Conditional`对 bean 进行有条件的注册
3. 使用`@Import`注解导入组件，有三种方式：
   1. 直接传入 bean 的类对象到`@Import`属性中。
   2. 实现`ImportSelector`接口，重写`selectImports()`方法，返回将要注册的bean 的全类名字符串数组，最后将这个实现类的类对象传入`@Import`属性中。
   3. 实现`ImportBeanDefinitionRegistrar`接口，手动注册 bean 到容器中。

4. 使用`FactoryBean`接口的实现类，即 bean 的工厂类，配置到配置类中。



# 4. 附录：常用注解

> [1.9. Annotation-based Container Configuration](https://docs.spring.io/spring-framework/docs/current/spring-framework-reference/core.html#beans-annotation-config)

| 注解                            | 说明                                                         |
| :------------------------------ | :----------------------------------------------------------- |
| @Controller                     | 组合注解（组合了@Component注解），应用在MVC层（控制层）,DispatcherServlet会自动扫描注解了此注解的类，然后将web请求映射到注解了@RequestMapping的方法上。 |
| @Service                        | 组合注解（组合了@Component注解），应用在service层（业务逻辑层） |
| @Reponsitory                    | 组合注解（组合了@Component注解），应用在dao层（数据访问层）  |
| @Component                      | 表示一个带注释的类是一个“组件”，成为[Spring](https://www.miaoroom.com/tag/spring)管理的Bean。当使用基于注解的配置和类路径扫描时，这些类被视为自动检测的候选对象。同时@Component还是一个元注解。 |
| @Autowired                      | [Spring](https://www.miaoroom.com/tag/spring)提供的工具（由Spring的依赖注入工具（BeanPostProcessor、BeanFactoryPostProcessor）自动注入。） |
| @Resource                       | JSR-250提供的注解                                            |
| @Inject                         | JSR-330提供的注解                                            |
| @Configuration                  | 声明当前类是一个配置类（相当于一个Spring配置的xml文件）      |
| @ComponentScan                  | 自动扫描指定包下所有使用@Service,@Component,@Controller,@Repository的类并注册 |
| @Bean                           | 注解在方法上，声明当前方法的返回值为一个Bean。返回的Bean对应的类中可以定义init()方法和destroy()方法，然后在@Bean(initMethod=“init”,destroyMethod=“destroy”)定义，在构造之后执行init，在销毁之前执行destroy。 |
| @Aspect                         | 声明一个切面（就是说这是一个额外功能）                       |
| @After                          | 后置建言（advice），在原方法前执行。                         |
| @Before                         | 前置建言（advice），在原方法后执行。                         |
| @Around                         | 环绕建言（advice），在原方法执行前执行，在原方法执行后再执行（@Around可以实现其他两种advice） |
| @PointCut                       | 声明切点，即定义拦截规则，确定有哪些方法会被切入             |
| @Transactional                  | 声明事务（一般默认配置即可满足要求，当然也可以自定义）       |
| @Cacheable                      | 声明数据缓存                                                 |
| @EnableAspectJAutoProxy         | 开启Spring对AspectJ的支持                                    |
| @Value                          | 值得注入。经常与Sping EL表达式语言一起使用，注入普通字符，系统属性，表达式运算结果，其他Bean的属性，文件内容，网址请求内容，配置文件属性值等等 |
| @PropertySource                 | 指定文件地址。提供了一种方便的、声明性的机制，用于向Spring的环境添加PropertySource。与@configuration类一起使用。 |
| @PostConstruct                  | 标注在方法上，该方法在构造函数执行完成之后执行。             |
| @PreDestroy                     | 标注在方法上，该方法在对象销毁之前执行。                     |
| @Profile                        | 表示当一个或多个指定的文件是活动的时，一个组件是有资格注册的。使用@Profile注解类或者方法，达到在不同情况下选择实例化不同的Bean。@Profile(“dev”)表示为dev时实例化。 |
| @EnableAsync                    | 开启异步任务支持。注解在配置类上。                           |
| @Async                          | 注解在方法上标示这是一个异步方法，在类上标示这个类所有的方法都是异步方法。 |
| @EnableScheduling               | 注解在配置类上，开启对计划任务的支持。                       |
| @Scheduled                      | 注解在方法上，声明该方法是计划任务。支持多种类型的计划任务：cron,fixDelay,fixRate |
| @Conditional                    | 根据满足某一特定条件创建特定的Bean                           |
| @Enable*                        | 通过简单的@Enable*来开启一项功能的支持。所有@Enable*注解都有一个@Import注解，@Import是用来导入配置类的，这也就意味着这些自动开启的实现其实是导入了一些自动配置的Bean(1.直接导入配置类2.依据条件选择配置类3.动态注册配置类) |
| @RunWith                        | 这个是Junit的注解，springboot集成了junit。一般在测试类里使用:@RunWith(SpringJUnit4ClassRunner.class) — SpringJUnit4ClassRunner在JUnit环境下提供Sprng TestContext Framework的功能 |
| @ContextConfiguration           | 用来加载配置ApplicationContext，其中classes属性用来加载配置类:@ContextConfiguration(classes = {TestConfig.class(自定义的一个配置类)}) |
| @ActiveProfiles                 | 用来声明活动的profile–@ActiveProfiles(“prod”(这个prod定义在配置类中)) |
| @EnableWebMvc                   | 用在配置类上，开启SpringMvc的Mvc的一些默认配置：如ViewResolver，MessageConverter等。同时在自己定制SpringMvc的相关配置时需要做到两点：1.配置类继承WebMvcConfigurerAdapter类2.就是必须使用这个@EnableWebMvc注解。 |
| @RequestMapping                 | 用来映射web请求（访问路径和参数），处理类和方法的。可以注解在类和方法上，注解在方法上的@RequestMapping路径会继承注解在类上的路径。同时支持Serlvet的request和response作为参数，也支持对request和response的媒体类型进行配置。其中有value(路径)，produces(定义返回的媒体类型和字符集)，method(指定请求方式)等属性。 |
| @GetMapping                     | GET方式的@RequestMapping                                     |
| @PostMapping                    | POST方式的@RequestMapping                                    |
| @ResponseBody                   | 将返回值放在response体内。返回的是数据而不是页面             |
| @RequestBody                    | 允许request的参数在request体中，而不是在直接链接在地址的后面。此注解放置在参数前。 |
| @PathVariable                   | 放置在参数前，用来接受路径参数。                             |
| @RestController                 | 组合注解，组合了@Controller和@ResponseBody,当我们只开发一个和页面交互数据的控制层的时候可以使用此注解。 |
| @ControllerAdvice               | 用在类上，声明一个控制器建言，它也组合了@Component注解，会自动注册为Spring的Bean。 |
| @ExceptionHandler               | 用在方法上定义全局处理，通过他的value属性可以过滤拦截的条件：@ExceptionHandler(value=Exception.class)–表示拦截所有的Exception。 |
| @ModelAttribute                 | 将键值对添加到全局，所有注解了@RequestMapping的方法可获得次键值对（就是在请求到达之前，往model里addAttribute一对name-value而已）。 |
| @InitBinder                     | 通过@InitBinder注解定制WebDataBinder（用在方法上，方法有一个WebDataBinder作为参数，用WebDataBinder在方法内定制数据绑定，例如可以忽略request传过来的参数Id等）。 |
| @WebAppConfiguration            | 一般用在测试上，注解在类上，用来声明加载的ApplicationContext是一个WebApplicationContext。他的属性指定的是Web资源的位置，默认为src/main/webapp,我们可以修改为：@WebAppConfiguration(“src/main/resources”)。 |
| @EnableAutoConfiguration        | 此注释自动载入应用程序所需的所有Bean——这依赖于Spring Boot在类路径中的查找。该注解组合了@Import注解，@Import注解导入了EnableAutoCofigurationImportSelector类，它使用SpringFactoriesLoader.loaderFactoryNames方法来扫描具有META-INF/spring.factories文件的jar包。而spring.factories里声明了有哪些自动配置。 |
| @SpingBootApplication           | SpringBoot的核心注解，主要目的是开启自动配置。它也是一个组合注解，主要组合了@Configurer，@EnableAutoConfiguration（核心）和@ComponentScan。可以通过@SpringBootApplication(exclude={想要关闭的自动配置的类名.class})来关闭特定的自动配置。 |
| @ImportResource                 | 虽然Spring提倡零配置，但是还是提供了对xml文件的支持，这个注解就是用来加载xml配置的。例：@ImportResource({"classpath |
| @ConfigurationProperties        | 将properties属性与一个Bean及其属性相关联，从而实现类型安全的配置。例：@ConfigurationProperties(prefix=“authot”，locations={"classpath |
| @ConditionalOnBean              | 条件注解。当容器里有指定Bean的条件下。                       |
| @ConditionalOnClass             | 条件注解。当类路径下有指定的类的条件下。                     |
| @ConditionalOnExpression        | 条件注解。基于SpEL表达式作为判断条件。                       |
| @ConditionalOnJava              | 条件注解。基于JVM版本作为判断条件。                          |
| @ConditionalOnJndi              | 条件注解。在JNDI存在的条件下查找指定的位置。                 |
| @ConditionalOnMissingBean       | 条件注解。当容器里没有指定Bean的情况下。                     |
| @ConditionalOnMissingClass      | 条件注解。当类路径下没有指定的类的情况下。                   |
| @ConditionalOnNotWebApplication | 条件注解。当前项目不是web项目的条件下。                      |
| @ConditionalOnResource          | 条件注解。类路径是否有指定的值。                             |
| @ConditionalOnSingleCandidate   | 条件注解。当指定Bean在容器中只有一个，后者虽然有多个但是指定首选的Bean。 |
| @ConditionalOnWebApplication    | 条件注解。当前项目是web项目的情况下。                        |
| @EnableConfigurationProperties  | 注解在类上，声明开启属性注入，使用@Autowired注入。例：@EnableConfigurationProperties(HttpEncodingProperties.class)。 |
| @AutoConfigureAfter             | 在指定的自动配置类之后再配置。例：@AutoConfigureAfter(WebMvcAutoConfiguration.class) |
