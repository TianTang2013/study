### Spring启动流程

----
[TOC]

----
#### 1. Demo创建
* `Demo`代码十分简单，整个工程结构如下:
* `pom`依赖
```xml4
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>5.1.8.RELEASE</version>
</dependency>
```
* service包下的两个类`OrderService`、`UserService`只加了`@Service`注解，dao包下的两个类`OrderDao`、`UserDao`只加了`@Repository`注解。`MainApplication`类中只写`main()`方法。代码如下：
```java
public static void main(String[] args) {
	AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);

    UserService userService = applicationContext.getBean(UserService.class);
    System.out.println(userService);

    OrderService orderService = applicationContext.getBean(OrderService.class);
    System.out.println(orderService);

    UserDao userDao = applicationContext.getBean(UserDao.class);
    System.out.println(userDao);

    OrderDao orderDao = applicationContext.getBean(OrderDao.class);
    System.out.println(orderDao);

    applicationContext.close();
}
```
* `AppConfig`类是一个配置类，类上加了两个注解，加`@Configuration`表明`AppConfig`是一个配置类，加`@ComponentScan`是为了告诉`Spring`要扫描哪些包，代码如下:
```java
@Configuration
@ComponentScan("com.tiantang.study")
public class AppConfig {
}
```
#### 2. 启动
* 运行`MainApplication`中的`main()`方法，这样一个`Spring`容器就运行起来了。控制台分别打印出了`UserService`、`OrderService`、`UserDao`、`OrderDao`的`hash`码。
* 那么问题来了，相比以往`xml`配置的方式，现在就这么几行简单的代码，一个`Spring`容器就能运行起来，我们就能从容器中获取到`Bean`，`Spring`内部是如何做到的呢？下面就来逐步分析`Spring`启动的源码。
#### 3. 入口
* 程序的入口为`main()`方法，从代码中可以发现，核心代码只有一行，`new AnnotationConfigApplicationContext(AppConfig.class)`，通过这一行代码，就将`Spring`容器给创建完成，然后我们就能通过`getBean()`从容器中获取到对象的了。因此，分析`Spring`源码，就从`AnnotationConfigApplicationContext`的有参构造函数开始。
* `AnnotationConfigApplicationContext`与`ClassPathXmlApplicationContext`作用一样，前者对应的是采用`JavaConfig`技术的应用，后者对应的是`XML`配置的应用 
#### 4. 基础概念
* 在进行`Spring`源码阅读之前，需要先理解几个概念。
* 1. `Spring`会将所有交由`Spring`管理的类，扫描其`class`文件，将其解析成`BeanDefinition`，在`BeanDefinition`中会描述类的信息，例如:这个类是否是单例的，`Bean`的类型，是否是懒加载，依赖哪些类，自动装配的模型。`Spring`创建对象时，就是根据`BeanDefinition`中的信息来创建`Bean`。
* 2. `Spring`容器在本文可以简单理解为`DefaultListableBeanFactory`,它是`BeanFactory`的实现类，这个类有几个非常重要的属性：`beanDefinitionMap`是一个`map`，用来存放`bean`所对应的`BeanDefinition`；`beanDefinitionNames`是一个`List`集合，用来存放所有`bean`的`name`；`singletonObjects`是一个`Map`，用来存放所有创建好的单例`Bean`。
* 3. `Spring`中有很多后置处理器，但最终可以分为两种，一种是`BeanFactoryPostProcessor`，一种是`BeanPostProcessor`。前者的用途是用来干预`BeanFactory`的创建过程，后者是用来干预`Bean`的创建过程。后置处理器的作用十分重要，`bean`的创建以及`AOP`的实现全部依赖后置处理器。
#### 5. AnnotationConfigApplicationContext的构造方法
* `AnnotationConfigApplicationContext`的构造函数的参数，是一个可变数组，可以传多个配置类，在本次`Demo`中，只传了`AppConfig`一个类。
* 在构造函数中，会先调用`this()`，在`this()`中通过调用父类构造器初始化了`BeanFactory`，以及向容器中注册了7个后置处理器。然后调用`register()`，将构造方法的参数放入到`BeanDefinitionMap`中。最后执行`refresh()`方法，这是整个`Spring`容器启动的核心，本文也将重点分析`refresh()`方法的流程和作用。
```java
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
	// 会初始化一个BeanFactory,为默认的DefaultListableBeanFactory
	// 会初始化一个beanDefinition的读取器，同时向容器中注册了7个spring的后置处理器(包括BeanPostProcessor和BeanFactoryPostProcessor)
	// 会初始化一个扫描器，后面似乎并没有用到这个扫描器，在refresh()中使用的是重新new的一个扫描器。
	this();
	// 将配置类注册进BeanDefinitionMap中
	register(annotatedClasses);
	refresh();
}
```
##### 5.1 this()调用
* `this()`会调用`AnnotationConfigApplicationContext`无参构造方法，而在`Java`的继承中，会先调用父类的构造方法。所以会先调用`AnnotationConfigApplicationContext`的父类`GeniricApplicationContext`的构造方法，在父类中初始化`beanFactory`，即直接`new`了一个`DefaultListableBeanFactory`。
```java
public GenericApplicationContext() {
	this.beanFactory = new DefaultListableBeanFactory();
}
```
* 在`this()`中通过`new AnnotatedBeanDefinitionReader(this)`实例化了一个`Bean`读取器，并向`BeanDefinitionMap`中添加了`7`个元素。通过`new ClassPathBeanDefinitionScanner(this)`实例化了一个扫描器(该扫描器在后面并没有用到)。
```java
public AnnotationConfigApplicationContext() {
    // 此处会先调用父类的构造器，即先执行 super(),初始化DefaultListableBeanFactory
    // 初始化了bean的读取器，并向spring中注册了7个spring自带的类，这里的注册指的是将这7个类对应的BeanDefinition放入到到BeanDefinitionMap中
    this.reader = new AnnotatedBeanDefinitionReader(this);
    // 初始化扫描器
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```
* 执行`this.reader = new AnnotatesBeanDefinitionReader(this)`时，最后会调用到`AnnotationConfigUtils.registerAnnotationConfigProcessors(BeanDefinitionRegistry registry,Object source)`方法，这个方法向`BeanDefinitionMap`中添加了`7`个类，这`7`个类的`BeanDefinition`(关于`BeanDefinition`的介绍可以参考前面的解释)均为`RootBeanDefinition`，这几个类分别为`ConfigurationClassPostProcessor`、`AutowiredAnnotationBeanPostProcessor`、`CommonAnnotationBeanPostProcessor`、`RequiredAnnotationBeanPostProcessor`、`PersistenceBeanPostProcessor`、`EventListenerMethodProcessor`、`DefaultEventListenerFactory`。
* 这7个类中，`ConfigurationClassPostProcessor`、`AutowiredAnnotationBeanPostProcessor`、`CommonAnnotationBeanPostProcessor`这三个类非常重要，这里先在下面代码中简单介绍了一下作用，后面会单独写文章分析它们的作用。本文的侧重点是先介绍完`Spring`启动的流程。
```java
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, @Nullable Object source) {
	DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
	// 省略部分代码 ...
	Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);
	// 注册ConfigurationClassPostProcessor,这个类超级重要，它完成了对加了Configuration注解类的解析，@ComponentScan、@Import的解析。
	if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
		RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
		def.setSource(source);
		beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
	}
	// 注册AutowiredAnnotationBeanPostProcessor,这个bean的后置处理器用来处理@Autowired的注入
	if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
		RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
		def.setSource(source);
		beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
	}
	// 注册RequiredAnnotationBeanPostProcessor
	if (!registry.containsBeanDefinition(REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
		RootBeanDefinition def = new RootBeanDefinition(RequiredAnnotationBeanPostProcessor.class);
		def.setSource(source);
		beanDefs.add(registerPostProcessor(registry, def, REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
	}
	// 注册CommonAnnotationBeanPostProcessor，用来处理如@Resource，@PostConstruct等符合JSR-250规范的注解
	// Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
	if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
		RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
		def.setSource(source);
		beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
	}
	// 注册PersistenceAnnotationBeanPostProcessor，来用支持JPA
	// Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
	if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
		RootBeanDefinition def = new RootBeanDefinition();
		try {
			def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
					AnnotationConfigUtils.class.getClassLoader()));
		}
		catch (ClassNotFoundException ex) {
			throw new IllegalStateException(
					"Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
		}
		def.setSource(source);
		beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
	}

	// 注册EventListenerMethodProcessor，用来处理方法上加了@EventListener注解的方法
	if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
		RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
		def.setSource(source);
		beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
	}

	// 注册DefaultEventListenerFactory，暂时不知道干啥用的，从类名来看，是一个事件监听器的工厂
	if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
		RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
		def.setSource(source);
		beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
	}
	return beanDefs;
}
```
* 调用`this.scanner = new ClassPathBeanDefinitionScanner(this)`来初始化一个扫描器，这个扫描器在后面扫描包的时候，并没有用到，猜测是`Spring`为了满足其他的场景而初始化的，例如: 开发人员手动通过`register(configClass)`时，扫描包时使用的。
##### 5.2 register(annotatedClasses)
> 将传入的配置类`annotatedClasses`解析成`BeanDefinition`(实际类型为`AnnotatedGenericBeanDefinition`)，然后放入到`BeanDefinitionMap`中，这样后面在`ConfigurationClassPostProcessor`中能解析`annotatedClasses`，例如`demo`中的`AppConfig`类，只有解析了`AppConfig`类，才能知道`Spring`要扫描哪些包(因为在`AppConfig`类中添加了`@ComponentScan`注解)，只有知道要扫描哪些包了，才能扫描出需要交给`Spring`管理的`bean`有哪些，这样才能利用`Spring`来创建`bean`。

##### 5.3 执行refresh()方法
> `refresh()`方法是整个`Spring`容器的核心，在这个方法中进行了`bean`的实例化、初始化、自动装配、`AOP`等功能。下面先看看`refresh()`方法的代码，代码中加了部分个人的理解，简单介绍了每一行代码作用，后面会针对几个重要的方法做出详细分析
```java
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
		// 初始化属性配置文件、检验必须属性以及监听器
		prepareRefresh();
		// Tell the subclass to refresh the internal bean factory.
		// 给beanFactory设置序列化id
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
		// 向beanFactory中注册了两个BeanPostProcessor,以及三个和环境相关的bean
		// 这两个后置处理器为ApplicationContextAwareProcessor和ApplicationListenerDetector
		// 前一个后置处理是为实现了ApplicationContextAware接口的类，回调setApplicationContext()方法，
		// 后一个处理器时用来检测ApplicationListener类的，当某个Bean实现了ApplicationListener接口的bean被创建好后，会被加入到监听器列表中
		prepareBeanFactory(beanFactory);
		try {
			// Allows post-processing of the bean factory in context subclasses.
			// 空方法，由子类实现
			postProcessBeanFactory(beanFactory);
			// 执行所有的BeanFactoryPostProcessor，包括自定义的，以及spring内置的。默认情况下，容器中只有一个BeanFactoryPostProcessor,即：Spring内置的，ConfigurationClassPostProcessor(这个类很重要)
			// 会先执行实现了BeanDefinitionRegistryPostProcessor接口的类，然后执行BeanFactoryPostProcessor的类
			// ConfigurationClassPostProcessor类的postProcessorBeanFactory()方法进行了@Configuration类的解析，@ComponentScan的扫描，以及@Import注解的处理
			// 经过这一步以后,会将所有交由spring管理的bean所对应的BeanDefinition放入到beanFactory的beanDefinitionMap中
			// 同时ConfigurationClassPostProcessor类的postProcessorBeanFactory()方法执行完后，向容器中添加了一个后置处理器————ImportAwareBeanPostProcessor
			invokeBeanFactoryPostProcessors(beanFactory);
			// 注册所有的BeanPostProcessor，因为在方法里面调用了getBean()方法，所以在这一步，实际上已经将所有的BeanPostProcessor实例化了
            // 为什么要在这一步就将BeanPostProcessor实例化呢？因为后面要实例化bean，而BeanPostProcessor是用来干预bean的创建过程的，所以必须在bean实例化之前就实例化所有的BeanPostProcessor(包括开发人员自己定义的)
			// 最后再重新注册了ApplicationListenerDetector，这样做的目的是为了将ApplicationListenerDetector放入到后置处理器的最末端
			registerBeanPostProcessors(beanFactory);
			// Initialize message source for this context.
           // 初始化MessageSource，用来做消息国际化。在一般项目中不会用到消息国际化
			initMessageSource();
			// Initialize event multicaster for this context.
			// 初始化事件广播器，如果容器中存在了名字为applicationEventMulticaster的广播器，则使用该广播器
			// 如果没有，则初始化一个SimpleApplicationEventMulticaster
            // 事件广播器的用途是，发布事件，并且为所发布的时间找到对应的事件监听器。
			initApplicationEventMulticaster();

			// Initialize other special beans in specific context subclasses.
			// 执行其他的初始化操作，例如和SpringMVC整合时，需要初始化一些其他的bean，但是对于纯spring工程来说，onFresh方法是一个空方法
			onRefresh();

			// Check for listener beans and register them.
			// 这一步会将自定义的listener的bean名称放入到事件广播器中
			// 同时还会将早期的ApplicationEvent发布(对于单独的spring工程来说，在此时不会有任何ApplicationEvent发布，但是和springMVC整合时，springMVC会执行onRefresh()方法，在这里会发布事件)
			registerListeners();
			// 实例化剩余的非懒加载的单例bean(注意：剩余、非懒加载、单例)
            // 为什么说是剩余呢？如果开发人员自定义了BeanPosrProcessor，而BeanPostProcessor在前面已经实例化了，所以在这里不会再实例化，因此这里使用剩余一词
			finishBeanFactoryInitialization(beanFactory);
			// 结束refresh，主要干了一件事，就是发布一个事件ContextRefreshEvent，通知大家spring容器refresh结束了。
			finishRefresh();
		}
		catch (BeansException ex) {
			// 出异常后销毁bean
			destroyBeans();
			// Reset 'active' flag.
			cancelRefresh(ex);
			// Propagate exception to caller.
			throw ex;
		}
		finally {
           // 在bean的实例化过程中，会缓存很多信息，例如bean的注解信息，但是当单例bean实例化完成后，这些缓存信息已经不会再使用了，所以可以释放这些内存资源了
			resetCommonCaches();
		}
	}
}
```
#### 6. refresh()方法
* 在`refresh()`方法中，比较重要的方法为`invokeBeanFactoryPostProcessors(beanFactory)` 和 `finishBeanFactoryInitialization(beanFactory)`。其他的方法相对而言比较简单，下面主要分析这两个方法，其他方法的作用，可以参考上面源码中的注释。

##### 6.1 invokeBeanFactoryPostProcessors()
* 该方法的作用是执行所有的`BeanFactoryPostProcessor`，由于`Spring`会内置一个`BeanFactoryPostProcessor`，即`ConfigurationClassPostProcessor`(如果开发人员不自定义，默认情况下只有这一个`BeanFactoryPostProcessor`)，这个后置处理器在处理时，会解析出所有交由`Spring`容器管理的`Bean`，将它们解析成`BeanDefinition`，然后放入到`BeanFactory`的`BeanDefinitionMap`中。
* 该方法最终会调用到`PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors()`方法，主要作用是执行所有`BeanFactoryPostProcessor`的`postProcessorBeanFactory()`方法。`BeanFactoryPostProcessor`又分为两种情况，一种是直接实现`BeanFactoryPostProcessor`接口的类，另一种情况是实现了`BeanDefinitionRegistryPostProcessor`接口(`BeanDefinitionRegistryPostProcessor`继承了`BeanFactoryPostProcessor`接口)。
* 图
* 在执行过程中先执行所有的`BeanDefinitionRegistryPostProcessor`的`postProcessorBeanDefinitionRegistry()`方法，然后再执行`BeanFacotryPostProcessor`的`postProcessorBeanFactory()`方法。
```java
public interface BeanFactoryPostProcessor {
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```
```java
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;
}
```
* 默认情况下，`Spring`有一个内置的`BeanFactoryPostProcessor`，即：`ConfigurationClassPostProcessor`类，该类实现了`BeanDefinitionRegistryPostProcessor`类，所以会执行`ConfigurationClassPostProcessor.postProcessorBeanDefinitionRegistry`,`ConfigurationClassPostProcessor`的`UML`图如上(删减了部分不重要的继承关系)

##### 6.2 registerBeanPostProcessors()
* 该方法的作用是找到所有的`BeanPostProcessor`，然后将这些`BeanPostProcessor`实例化(会调用`getBean()`方法，`getBean()`方法的主要逻辑是，如果`bean`存在于`BeanFactory`中，则返回`bean`；如果不存在，则会去创建。在后面会仔细分析`getBean()`的执行逻辑)。将这些`PostProcessor`实例化后，最后放入到`BeanFactory`的`beanPostProcessors`属性中。
* 问题：如何找到所有的`BeanPostProcessor`? 包括`Spring`内置的和开发人员自定义的。
* 由于在`refresh()`方法中，会先执行完`invokeBeanFactoryPostProcessor()`方法，这样所有自定义的`BeanPostProcessor`类均已经被扫描出并解析成`BeanDefinition`(扫描和解析又是谁做的呢？`ConfigurationClassPostProcessor`做的)，存入至`BeanFactory`的`BeanDefinitionMap`，所以这儿能通过方法如下一行代码找出所有的`BeanPostProcessor`，然后通过`getBean()`全部实例化，最后再将实例化后的对象加入到`BeanFactory`的`beanPostProcessors`属性中，该属性是一个`List`集合。
```java
String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
```
* 最后再重新注册了`ApplicationListenerDetector`，这样做的目的是为了将`ApplicationListenerDetector`放入到后置处理器的最末端
* `registerBeanPostProcessor()` 最终调用的是`PostProcessorRegistrationDelegate.registerBeanPostProcessors()`，下面是`PostProcessorRegistrationDelegate.registerBeanPostProcessors()`方法的代码
```java
public static void registerBeanPostProcessors(
		ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

	// 从BeanDefinitionMap中找出所有的BeanPostProcessor
	String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);
	// ... 省略部分代码 ...

	// 分别找出实现了PriorityOrdered、Ordered接口以及普通的BeanPostProcessor
	for (String ppName : postProcessorNames) {
		if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
			// 此处调用了getBean()方法，因此在此处就会实例化出BeanPostProcessor
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			priorityOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
			orderedPostProcessorNames.add(ppName);
		}
		else {
			nonOrderedPostProcessorNames.add(ppName);
		}
	}

	// First, register the BeanPostProcessors that implement PriorityOrdered.
	sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
	// 将实现了PriorityOrdered接口的BeanPostProcessor添加到BeanFactory的beanPostProcessors集合中
	registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

	// 下面这部分代码与上面的代码逻辑一致，是将实现了Ordered接口以及普通的BeanPostProcessor实例化以及添加到beanPostProcessors结合中，逻辑与处理PriorityOrdered的后置处理器一样
	List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
	for (String ppName : orderedPostProcessorNames) {
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		orderedPostProcessors.add(pp);
		if (pp instanceof MergedBeanDefinitionPostProcessor) {
			internalPostProcessors.add(pp);
		}
	}
	sortPostProcessors(orderedPostProcessors, beanFactory);
	registerBeanPostProcessors(beanFactory, orderedPostProcessors);
	
	// Now, register all regular BeanPostProcessors.
	List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
	for (String ppName : nonOrderedPostProcessorNames) {
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		nonOrderedPostProcessors.add(pp);
		if (pp instanceof MergedBeanDefinitionPostProcessor) {
			internalPostProcessors.add(pp);
		}
	}
	registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);
	
	// Finally, re-register all internal BeanPostProcessors.
	sortPostProcessors(internalPostProcessors, beanFactory);
	registerBeanPostProcessors(beanFactory, internalPostProcessors);
	
	// Re-register post-processor for detecting inner beans as ApplicationListeners,
	// moving it to the end of the processor chain (for picking up proxies etc).
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));

	// 最后将ApplicationListenerDetector这个后置处理器一样重新放入到beanPostProcessor中，这样做的目的是为了将其放入到后置处理器的最末端
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
```
* 从上面的源码中可以发现，`BeanPostProcessor`存在优先级，实现了`PriorityOrdered`接口的优先级最高，其次是`Ordered`接口，最后是普通的`BeanPostProcessor`。优先级最高的，会最先放入到`beanPostProcessors`这个集合的最前面，这样在执行时，会最先执行优先级最高的后置处理器(因为`List`集合是有序的)。
* 这样在实际应用中，如果我们碰到需要优先让某个`BeanPostProcessor`执行，则可以让其实现`PriorityOrdered`接口或者`Ordered`接口。

##### 6.3 initMessageSource()
* 用来支持消息国际化，现在一般项目中不会用到国际化相关的知识。

##### 6.4 initApplicationEventMulticaster()
> 该方法初始化了一个事件广播器，如果容器中存在了`beanName`为`applicationEventMulticaster`的广播器，则使用该广播器；如果没有，则初始化一个`SimpleApplicationEventMulticaster`。该事件广播器是用来做应用事件分发的，这个类会持有所有的事件监听器(`ApplicationListener`)，当有`ApplicationEvent`事件发布时，该事件监听器能根据事件类型，检索到对该事件感兴趣的`ApplicationListener`。

* `initApplicationEventMulticaster()`方法的源码如下(省略了部分日志信息):
```java
protected void initApplicationEventMulticaster() {
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	// 判断spring容器中是否已经存在beanName = applicationEventMulticaster的事件广播器
	// 例如：如果开发人员自己注册了一个
	// 如果存在，则使用已经存在的；否则使用spring默认的:SimpleApplicationEventMulticaster
	if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
		this.applicationEventMulticaster =
				beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
	}
	else {
		this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
		beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
	}
}
```
##### 6.5 onRefresh()
> 执行其他的初始化操作，例如和`SpringMVC`整合时，需要初始化一些其他的`bean`，但是对于纯`Spring`工程来说，`onRefresh()`方法是一个空方法。

##### 6.6 registerListeners()
> 这一步会将自定义的`listener`的`bean`名称放入到事件广播器中,同时还会将早期的`ApplicationEvent`发布(对于单独的`Spring`工程来说，在此时不会有任何`ApplicationEvent`发布，但是和`SpringMVC`整合时，`SpringMVC`会执行`onRefresh()`方法，在这里会发布事件)。方法源码如下:
```java
protected void registerListeners() {
	// Register statically specified listeners first.
	for (ApplicationListener<?> listener : getApplicationListeners()) {
  getApplicationEventMulticaster().addApplicationListener(listener);
	}

	// 从BeanFactory中找到所有的ApplicationListener，但是不会进行初始化，因为需要在后面bean实例化的过程中，让所有的BeanPostProcessor去改造它们
	String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
	for (String listenerBeanName : listenerBeanNames) {
		// 将事件监听器的beanName放入到事件广播器中
		getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
	}

	// 发布早期的事件(纯的spring工程，在此时一个事件都没有)
	Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
	this.earlyApplicationEvents = null;
	if (earlyEventsToProcess != null) {
		for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
			getApplicationEventMulticaster().multicastEvent(earlyEvent);
		}
	}
}
```
##### 6.7 finishBeanFactoryInitialization()
> 该方法十分重要，它完成了所有非懒加载的单例`Bean`的实例化和初始化，属性的填充以及解决了循环依赖等问题。

* (注意这里特意将对象的实例化和初始化过程分开了，因为在`Spring`创建`Bean`的过程中，是先将`Bean`通过反射创建对象，然后通过后置处理器(`BeanPostProcessor`)来为对象的属性赋值。所以这里的实例化时指将`Bean`创建出来，初始化是指为`bean`的属性赋值)。
* `finishBeanFactoryInitialization()`方法的代码如下，`bean`的创建和初始化均在`beanFactory.preInstantiateSingletons()`中实现。
```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
	// 初始化转换服务
	if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
			beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
		beanFactory.setConversionService(
				beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
	}
	// 如果前面没有注册一个类似于PropertyPlaceholderConfigurer后置处理器的bean，那么在这儿会注册一个内置的属性后置处理器
	// 这儿主要是处理被加了注解的属性
	if (!beanFactory.hasEmbeddedValueResolver()) {
		beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
	}
	// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
	String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
	for (String weaverAwareName : weaverAwareNames) {
		getBean(weaverAwareName);
	}
	beanFactory.setTempClassLoader(null);
	// 将BeanFactory的configurationFrozen属性设置为true,给frozenBeanDefinitionNames属性赋值
	// 目的是为了不让在其他的地方在修改bean的BeanDefinition
	beanFactory.freezeConfiguration();

	// Instantiate all remaining (non-lazy-init) singletons.
	// 实例化剩下所有的非懒加载的单例
	beanFactory.preInstantiateSingletons();
}
```
######  6.7.1 preInstantiateSingletons()方法
 * 该方法会先判断`bean`是否是一个`FactoryBean`，以及是否立即实例化`FactoryBean`的`getObject()`返回的对象，但最终均是调用`getBean()`方法去实例化对象。在`Bean`实例化、初始化完成后，会判断`Bean`是否实现了`SmartSingletonInitializing`接口，如果实现了，则会调用该接口的`afterSingletonInstantiated()`方法。
 * `Tips`：这里提到了`FactoryBean`,不是`BeanFactory`。这两者名字很像，但作用却是天差地别，有兴趣的朋友可以先自己`Google`查下相关知识。这里先简单介绍一下，后续会单独写一篇文章介绍`FactoryBean`，并通过`FactoryBean`去解析`Spring`与`MyBatis`整合的原理。
 * `FactoryBean`是一个接口，该接口的实现类会向容器中注册两个`bean`，一个是实现类本身所代表类型的对象，一个是通过重写`FactoryBean`接口中`getObject()`方法所返回的`bean`。如下例子中：会向容器中注册两个`bean`，一个是`MapperFactoryBean`本身，一个是`UserMapper`。
```java
@Component
public class MapperFactoryBean implements FactoryBean {
    @Override
    public Object getObject() throws Exception {
        return new UserMapper();
    }

    @Override
    public Class<?> getObjectType() {
        return UserMapper.class;
    }
}
```
* 由于`bean`的实例化过程太过复杂，后面会结合流程图去分析源码。`preInstantiatedSingletons()`方法的执行流程图如下
* 图
* `preInstantiatedSingletons()`代码如下
```java
public void preInstantiateSingletons() throws BeansException {
	if (logger.isDebugEnabled()) {
		logger.debug("Pre-instantiating singletons in " + this);
	}
	List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);
	// Trigger initialization of all non-lazy singleton beans...
	for (String beanName : beanNames) {
		RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
		if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
			// 判断是否是factoryBean,如果是FactoryBean，则进行FactoryBean原生的实例化(非getObject()方法对应的对象)。
			// 还需要判断它是否立即实例化getObject()返回的对象，根据SmartFactoryBean的isEagerInit()的返回值判断是否需要立即实例化
			if (isFactoryBean(beanName)) {
				// 首先实例化BeanFactory的原生对象，然后再根据isEagerInit()判断是否实例化BeanFactory中getObject()返回的类型的对象
				Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
				if (bean instanceof FactoryBean) {
					final FactoryBean<?> factory = (FactoryBean<?>) bean;
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
										((SmartFactoryBean<?>) factory)::isEagerInit,
								getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					// 如果isEagerInit为true,则立即实例化FactoryBean所返回的类型的对象
					if (isEagerInit) {
						getBean(beanName);
					}
				}
			}
			else {
				getBean(beanName);
			}
		}
	}
	// 在bean实例化以及属性赋值完成后，如果bean实现了SmartInitializingSingleton接口，则回调该接口的方法
	for (String beanName : beanNames) {
		Object singletonInstance = getSingleton(beanName);
		if (singletonInstance instanceof SmartInitializingSingleton) {
			final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
			if (System.getSecurityManager() != null) {
				AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
					smartSingleton.afterSingletonsInstantiated();
					return null;
				}, getAccessControlContext());
			}
			else {
				smartSingleton.afterSingletonsInstantiated();
			}
		}
	}
}
```
* 从上面源码中发现，不论是`FactoryBean`还是普通`Bean`，最终都是调用`getBean()`方法去创建`bean`。

###### 6.7.2 getBean()
* `getBean()`方法会调用`doGetBean()`方法。
```java
public Object getBean(String name) throws BeansException {
	return doGetBean(name, null, null, false);
}
```
###### 6.7.3 doGetBean()
* 在`doGetBean()`方法当中，会先从缓存中获取(即从`singletonObjects`这个`map`集合中获取，为什么要先从缓存中获取呢？因为要从`Spring`容器获取对象和创建对象，都是通过`getBean()`方法，对于单例对象而言，对象只会被创建一次，那么先从缓存中获取对象，如果存在，则不用去新创建了，这样就保证了单例对象只被创建一次)。如果缓存中存在，则接着调用`getObjectForBeanInstance()`方法，然后返回`bean`。如果缓存中不存在，则继续往下执行。
* 然后获取到`Bean`所对应的`BeanDefinition`对象。接着判断`bean`有没有依赖，`String[] dependsOn = mbd.getDependsOn()`，如果有依赖的对象，那么会先去实例化依赖的对象。
* 依赖的对象创建完成后，会调用`getSingleton(beanName,lambda)`方法，这个方法的第二个参数是一个`lambda`表达式，真正创建`bean`的逻辑是在表达式的方法体中，即`createBean()`方法，`createBean()`方法会创建完成`bean`，然后在`getSingleton(beanName,lambda)`方法中会将创建完成的`bean`存入到`singletonObjects`属性中。`createBean()`后面分析。
* 在上一步创建完`bean`后，最终仍会调用`getObjectForBeanInstance()`。这个方法的逻辑比较简单，先判断`bean`是否是一个`FactoryBean`，若不是，则直接返回`bean`；若是，则再判断`beanName`是否是以&符号开头，如果是，表示获取的是`FactoryBean`的原生对象，则直接返回`bean`；若不是以&符号开头，则会返回`FactoryBean`的`getObject()`方法的返回值对象。
* `doGetBean()`方法代码如下
```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
		@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
	final String beanName = transformedBeanName(name);
	Object bean;
	// Eagerly check singleton cache for manually registered singletons.
	Object sharedInstance = getSingleton(beanName);
	if (sharedInstance != null && args == null) {
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}
	else {
		// Fail if we're already creating this bean instance:
		// We're assumably within a circular reference.
		if (isPrototypeCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}
		// Check if bean definition exists in this factory.
		BeanFactory parentBeanFactory = getParentBeanFactory();
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			// Not found -> check parent.
			String nameToLookup = originalBeanName(name);
			if (parentBeanFactory instanceof AbstractBeanFactory) {
				return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
						nameToLookup, requiredType, args, typeCheckOnly);
			}
			else if (args != null) {
				// Delegation to parent with explicit args.
				return (T) parentBeanFactory.getBean(nameToLookup, args);
			}
			else {
				return parentBeanFactory.getBean(nameToLookup, requiredType);
			}
		}
		if (!typeCheckOnly) {
			// 标记bean为已创建
			// 并清除beanDefinition的缓存(mergedBeanDefinitions)
			markBeanAsCreated(beanName);
		}
		try {
			final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
			// 检查bean是否是抽象类
			checkMergedBeanDefinition(mbd, beanName, args);
			// Guarantee initialization of beans that the current bean depends on.
			// 保证当前bean所依赖的bean初始化
			String[] dependsOn = mbd.getDependsOn();
			if (dependsOn != null) {
				for (String dep : dependsOn) {
					// isDependent()方法用来判断dep是否依赖beanName
					if (isDependent(beanName, dep)) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
					}
					// 保存下依赖关系
					registerDependentBean(dep, beanName);
					try {
						getBean(dep);
					}
					catch (NoSuchBeanDefinitionException ex) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
					}
				}
			}
			// Create bean instance.
			if (mbd.isSingleton()) {
				// 此时在getSingleton方法中传入了一个lambda表达式，
				// 此时不会立即执行lambda表达式，而是在调用这个lambda表达式的getObject()方法时才开始执行lambda的方法体
				sharedInstance = getSingleton(beanName, () -> {
					try {
						return createBean(beanName, mbd, args);
					}
					catch (BeansException ex) {
						destroySingleton(beanName);
						throw ex;
					}
				});
				bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
			}

			else if (mbd.isPrototype()) {
				// It's a prototype -> create a new instance.
				Object prototypeInstance = null;
				try {
					beforePrototypeCreation(beanName);
					prototypeInstance = createBean(beanName, mbd, args);
				}
				finally {
					afterPrototypeCreation(beanName);
				}
				bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
			}
			else {
				String scopeName = mbd.getScope();
				final Scope scope = this.scopes.get(scopeName);
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
				}
				try {
					Object scopedInstance = scope.get(beanName, () -> {
						beforePrototypeCreation(beanName);
						try {
							return createBean(beanName, mbd, args);
						}
						finally {
							afterPrototypeCreation(beanName);
						}
					});
					bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
				}
				catch (IllegalStateException ex) {
					throw new BeanCreationException(beanName,
							"Scope '" + scopeName + "' is not active for the current thread; consider " +
							"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
							ex);
				}
			}
		}
		catch (BeansException ex) {
			cleanupAfterBeanCreationFailure(beanName);
			throw ex;
		}
	}
	if (requiredType != null && !requiredType.isInstance(bean)) {
		try {
			T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
			if (convertedBean == null) {
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
			return convertedBean;
		}
		catch (TypeMismatchException ex) {
			throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
		}
	}
	return (T) bean;
}
```
* `getObjectForBeanInstance()`方法的作用是为了根据`beanName`来判断是返回`FactoryBean`原生对象还是`getObject()`方法所返回的对象.若`beanName`以&符号开头，则表示返回`FactoryBean`原生对象，否则返回`getObject()`方法所返回的对象。
```java
protected Object getObjectForBeanInstance(
		Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {
	if (BeanFactoryUtils.isFactoryDereference(name)) {
		if (beanInstance instanceof NullBean) {
			return beanInstance;
		}
		if (!(beanInstance instanceof FactoryBean)) {
			throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
		}
	}

	// 如果不是一个FactoryBean对象或者是获取FactoryBean的原生对象(原生对象指的是beanName是以&开头)
	// 此时可以直接返回bean
	if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
		return beanInstance;
	}

	// 如果是获取FactoryBean的getObject()方法返回的类型对象，则需要进入到如下逻辑
	// 对于getObject()方法，它返回的对象是在在第一次调用getObject方法时进行实例化的，实例化完成以后，会将结果缓存在factoryBeanObjectCache中
	Object object = null;
	if (mbd == null) {
		object = getCachedObjectForFactoryBean(beanName);
	}
	if (object == null) {
		// Return bean instance from factory.
		FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
		// Caches object obtained from FactoryBean if it is a singleton.
		if (mbd == null && containsBeanDefinition(beanName)) {
			mbd = getMergedLocalBeanDefinition(beanName);
		}
		boolean synthetic = (mbd != null && mbd.isSynthetic());
		// 获取FactoryBean返回的对象
		object = getObjectFromFactoryBean(factory, beanName, !synthetic);
	}
	return object;
}
```
###### 6.7.4 createBean()
* `doGetBean()`最终会调用`createBean()`来创建`bean`。
* `createBean()`方法的代码中，主要有两行核心代码：
```java
Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
Object beanInstance = doCreateBean(beanName, mbdToUse, args);
```
* `resolveBeforeInstantiation()`方法在`bean`实例化之前调用，在这个方法中执行了后置处理器`InstantiationAwareBeanPostProcessor`的`postProcessBeforeInstantiation()`方法，在`bean`实例化之前对`bean`进行处理。这个扩展点的意义十分重大，`Spring`的`AOP`就是在这儿实现的，感兴趣的朋友可阅读`AnnotationAwareAspectJAutoProxyCreator`这个类的源码，后续会单独写一篇文章进行分析。
* 如果`resolveBeforeInstantiation()`的返回值不为`null`，则直接将结果返回。如果为`null`，则会继续执行方法`doCreateBean()`。在`doCreateBean()`方法中，进行了`Bean`的实例化、属性赋值、初始化等操作。
* `createBean()`方法的流程图
* 图
* `createBean()`方法的源代码
```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
		throws BeanCreationException {
	RootBeanDefinition mbdToUse = mbd;
	// Make sure bean class is actually resolved at this point, and
	// clone the bean definition in case of a dynamically resolved Class
	// which cannot be stored in the shared merged bean definition.
	Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
	if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
		mbdToUse = new RootBeanDefinition(mbd);
		mbdToUse.setBeanClass(resolvedClass);
	}
	// Prepare method overrides.
	try {
		mbdToUse.prepareMethodOverrides();
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
				beanName, "Validation of method overrides failed", ex);
	}
	try {
		// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
		// 第一次调用后置处理器(执行所有InstantiationAwareBeanPostProcessor的子类)
		// 如果InstantiationAwareBeanPostProcessor的子类的postProcessBeforeInstantiation()方法返回值不为空，表示bean需要被增强，
		// 此时将不会执行后面的逻辑，AOP的实际应用就是在这儿实现的
		Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
		if (bean != null) {
			return bean;
		}
	}
	catch (Throwable ex) {
		throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
				"BeanPostProcessor before instantiation of bean failed", ex);
	}

	try {
		// 第二次执行后置处理器的入口
		Object beanInstance = doCreateBean(beanName, mbdToUse, args);
		if (logger.isDebugEnabled()) {
			logger.debug("Finished creating instance of bean '" + beanName + "'");
		}
		return beanInstance;
	}
	catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
		throw ex;
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
	}
}
```
###### 6.7.5 doCreateBean()
* `doCreateBean()`方法会通过反射进行`Bean`的创建，然后对`bean`进行属性填充(在填充属性的同时，解决了循环依赖的问题)，最后会对`Bean`回调初始化相关的方法，例如：`BeanPostProcessor.postProcessBeforeInilization(),InilizaingBean.afterPropertiesSet()`,给`bean`配置的`initMethod()`方法，以及`BeanPostProcessor.postProcessAfterInilization()`。
* `doCreateBean()`执行的流程图如下：
* 图
* `doCreateBean()`方法的代码(删减了部分代码)如下:
```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
		throws BeanCreationException {
	// Instantiate the bean.
	BeanWrapper instanceWrapper = null;
	if (mbd.isSingleton()) {
		instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
	}
	if (instanceWrapper == null) {
		// 实例化bean(第二次执行后置处理器的入口),第二次执行后置处理器，主要是为了推断出实例化Bean所需要的构造器
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
	final Object bean = instanceWrapper.getWrappedInstance();
	Class<?> beanType = instanceWrapper.getWrappedClass();
	if (beanType != NullBean.class) {
		mbd.resolvedTargetType = beanType;
	}
	// 此时bean对象已经创建成功，但是没有设置属性和经过其他后置处理器处理
	// Allow post-processors to modify the merged bean definition.
	synchronized (mbd.postProcessingLock) {
		if (!mbd.postProcessed) {
			try {
				// 第三次执行后置处理器，缓存bean的注解元数据信息(用于后面在进行属性填充时使用)
				// 这一步对于CommonAnnotationBeanPostProcessor、AutowiredAnnotationBeanPostProcessor、RequiredAnnotationBeanPostProcessor这一类处理器
				// 主要是将bean的注解信息解析出来，然后缓存到后置处理器中的injectionMetadataCache属性中
				// 而对于ApplicationListenerDetector处理器，而是将bean是否是单例的标识存于singletonNames这个Map类型的属性中
				applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
			}
			mbd.postProcessed = true;
		}
	}
	// 判断一个bean是否放入到singletonFactories中(提前暴露出来，可以解决循环依赖的问题)
	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
			isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
		// 第四次出现后置处理器
		// 获取提前暴露的对象，可以解决循环引用的问题，实际上提前暴露出来的bean是放入到了singletonFactories中，key是beanName,value是一个lambda表达式
		addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
	}
	// Initialize the bean instance.
	Object exposedObject = bean;
	try {
		// 填充属性，第五次、第六次后置处理器入口
		populateBean(beanName, mbd, instanceWrapper);
		// 第七次、第八次执行后置处理器入口
		exposedObject = initializeBean(beanName, exposedObject, mbd);
	}
	catch (Throwable ex) {
	}
	return exposedObject;
}
```
* `createBeanInstance()`会通过反射创建对象时，会先执行后置处理器，通过调用后置处理器的`deternineCondidateConstructors()`方法来推断出使用哪一个构造器来创建`Bean`，典型的代表类有`AutowiredAnnotationBeanPostProcessor`。
* `bean`被通过反射创建完成后，会再次调用后置处理器`MergedBeanDefinitionPostProcessor.postProcessMargedBeanDefinition()`方法，这一步执行后置处理器的目的是为了找出加了`@Autowired`、`@Resource`等注解的属性和方法，然后将这些注解信息缓存到`injectionMetadataCache`属性中，便于后面在`bean`初始化阶段(属性赋值阶段)，根据`@Autowired`等注解实现自动装配。这一步的代表后置处理器有`AutowiredAnnotationBeanPostProcessor`、`CommonAnnotationBeanPostProcessor`
> `AutowiredAnnotationBeanPostProcessor`是用来处理`Spring`提供的注解和`JSR-330`中的部分注解，如：`@Autowired`，`@Value`，`@Inject`。`CommonAnnotationBeanPostProcessor`是用来处理`JSR-250`中的注解，如`@Resource`、`@PostConstruct`、`@PreDestroy`。

* 接下来会将半成品的`bean`(因为此时还未给`bean`的属性赋值，未完成自动装配，因此称之为半成品)放入到`DefaultSingletonBeanRegistry`类的`singletonFactories`的属性中，`singletonFactories`属性是一个`Map`，`key`为`beanName`,值为`ObjectFactory`类型(实际上就是一个`lambda`表达式)，当调用`ObjectFactory`的`getObject()`方法时，会执行`lambda`表达式的方法体，在当前场景下，`lambda`表达式的代码如下，实际上就是执行了一次`Bean`的后置处理器。这一步的目的是为了解决`bean`之间的循环依赖，究竟是如何解决循环依赖的，以后分析。
```java
protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
	Object exposedObject = bean;
	if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
				SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
                // 调用后置处理的方法获取bean早期暴露出来的bean对象(半成品)
				exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
			}
		}
	}
	return exposedObject;
}
```
* 然后会执行到`populateBean()`方法，在该方法中又会执行两次`Bean`后置处理器，第一次执行后置处理器是为了判断`Bean`是否需要继续填充属性，如果`InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()`方法返回的`false`，则表示不进行属性填充，`bean`就不会进行`@Autowired`等自动装配过程，`populateBean()`方法会直接结束。若返回`true`，则会进行接下来的属性填充，即会执行第二次后置处理器，`InstantiationAwareBeanPostProcessor.postProcessPropertyValue()`方法，这一步的主角就是`AutowiredAnnotationBeanPostProcessor`和`CommonAnnotationBeanPostBeanPostProcessor`了，它们会根据前面缓存在`injectionMetadataCache`中的注解信息来进行自动装配。
* 在执行完`populateBean()`方法后后，接下来回执行`initializeBean()`方法，即进入初始化阶段。在`initializeBean()`方法中，最先执行`invokeAwareMethods()`方法，即执行`Aware`接口的方法，如：`BeanNameAware`、`BeanClassLoaderAware`、`BeanFactoryAware`。然后再一次执行所有`Bean`后置处理器的`BeanPostProcessor.postProcessBeforeInitialization()`方法。接着执行`invokeInitMethods()`方法，在`invokeInitMethods()`方法中，会执行`InitializingBean`的`afterPropertiesSet()`方法，和定义`bean`时自定义的`initMethod()`方法。最后再一次执行`bean`后置处理器，`BeanPostProcessor.postProcessAfterInitialization()`。
* 至此`bean`的实例化、初始化过程已经完成，创建好的`bean`会被返回，若是单例`bean`，最后会被存放到`DefaultSingletonBeanRegistry`的`singletonObjects`中。
##### 6.8 finishRefresh()
* 执行到这一步，`Spring`容器的启动基本结束了，此时`Bean`已经被实例化完成，且完成了自动装配。执行`finishRefresh()`方法，是为了在容器`refresh()`结束时，做一些其他的操作，例如：发布`ContextRefreshedEvent`事件，这样当我们想在容器`refresh`完成后执行一些特殊的逻辑，就可以通过监听`ContextRefreshedEvent`事件来实现。`Spring`内置了四个和应用上下文(`ApplicationContextEvent`)有关的事件：`ContextRefreshedEvent`、`ContextStartedEvent`、`ContextStopedEvent`、`ContextClosedEvent`。
```java
protected void finishRefresh() {
    clearResourceCaches();
    initLifecycleProcessor();
    getLifecycleProcessor().onRefresh();
    // 发布ContextRefreshedEvent
    publishEvent(new ContextRefreshedEvent(this));
    LiveBeansView.registerApplicationContext(this˛);
}
```
##### 6.9 resetCommonCaches()
> 最后在`refresh()`方法的`finally`语句块中，执行了`resetCommonCaches()`方法。因为在前面创建`bean`时，对单例`bean`的元数据信息进行了缓存，而单例`bean`在容器启动后，不会再进行创建了，因此这些缓存的信息已经没有任何用处了，在这里进行清空，释放部分内存。

```java
protected void resetCommonCaches() {
    ReflectionUtils.clearCache();
    AnnotationUtils.clearCache();
    ResolvableType.clearCache();
    CachedIntrospectionResults.clearClassLoader(getClassLoader());
}
```

#### 7. Bean生命周期
* 从上面`Spring`的源码分析中，可以看出，启动过程中，`bean`的创建过程最为复杂，在创建过程中，前后一共出现了8次调用`BeanPostPorcessor`(实际上在`bean`的整个生命周期中，一共会出现`9`次调用后置处理器，第九次出现在`bean`的销毁阶段。)
* 结合`Spring`的源码，单例`bean`的生命周期可以总结为如下一张图
* 图
#### 8. 总结
* 本文介绍了`Spring`的启动流程，通过`AnnotationConfigApplicationContext`的有参构造方法入手，重点分析了`this()`方法和`refresh()`方法。在`this()`中初始化了一个`BeanFactory`，即`DefaultListableBeanFactory`；然后向容器中添加了7个内置的`bean`，其中就包括`ConfigurationClassPostProcessor`。
* 在`refresh()`方法中，又重点分析了`invokeBeanFactoryPostProcessor()`方法和`finishBeanFactoryInitialization()`方法。
* 在`invokeBeanFactoryPostProcessor()`方法中，通过`ConfigurationClassPostProcessor`类扫描出了所有交给`Spring`管理的类，并将`class`文件解析成对应的`BeanDefinition`。
* 在`finishBeanFactoryInitialization()`方法中，完成了非懒加载的单例`Bean`的实例化和初始化操作，主要流程为`getBean()` ——>`doGetBean()`——>`createBean()`——>`doCreateBean()`。在`bean`的创建过程中，一共出现了`8`次`BeanPostProcessor`的执行，在这些后置处理器的执行过程中，完成了`AOP`的实现、`bean`的自动装配、属性赋值等操作。
* 最后通过一张流程图，总结了`Spring`中单例`Bean`的生命周期。
#### 9. 计划
> 本文主要介绍了`Spring`的启动流程，但对于一些地方的具体实现细节没有展开分析，因此后续Spring源码分析的计划如下:

- [ ] `ConfigurationClassPostProcessor`类如何扫描包，解析配置类。
- [ ] `@Import`注解作用与`@Enable`系列注解的实现原理
- [ ] `JDK`动态代理与`CGLIB`代理
- [ ] `FactoryBean`的用途和源码分析
- [ ] `AutowiredAnnotationBeanPostProcessor`、`CommonAnnotationBeanPostProcessor`如何实现自动装配,`Spring`如何解决循环依赖
- [ ] `AOP`的实现原理
- [ ] `SpringBoot`源码分析

#### 10. 推荐性能监控工具
* 最后推荐一款开源的性能监控工具——`Pepper-Metrics`
> 地址： https://github.com/zrbcool/pepper-metrics
[GitHub](https://github.com/zrbcool/pepper-metrics)
> `Pepper-Metrics`是坐我对面的两位同事一起开发的开源组件，主要功能是通过比较轻量的方式与常用开源组件（`jedis/mybatis/motan/dubbo/servlet`）集成，收集并计算`metrics`，并支持输出到日志及转换成多种时序数据库兼容数据格式，配套的`grafana dashboard`友好的进行展示。项目当中原理文档齐全，且全部基于`SPI`设计的可扩展式架构，方便的开发新插件。另有一个基于`docker-compose`的独立`demo`项目可以快速启动一套`demo`示例查看效果`https://github.com/zrbcool/pepper-metrics-demo`。如果大家觉得有用的话，麻烦给个`star`，也欢迎大家参与开发，谢谢：）

