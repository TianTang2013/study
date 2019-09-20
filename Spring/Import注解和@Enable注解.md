> 点击上方`菜鸟飞呀飞`，即可关注微信公众号。

[TOC]
## 1. @Import注解
> 通过Import注解，我们有三种方式可以向Spring容器中注册Bean。相当于Spring中XML的<import/>标签。

### 1.1 直接注册
* 例如：@Import(RegularBean.class)。（RegularBean是开发人员自定义的一个类）。代码如下，在代码中通过在AppConfig类上加了一行注解：`@Import(RegularBean.class)`，这样就能从容器中获取到RegularBean的实例对象的。
```java
/**
* 自定义的一个普通类
*/
public class RegularBean {
}
```
```java
/**
* 在配置类上通过Import注解向Spring容器中注册RegularBean
*/
@Configuration
@Import(RegularBean.class)
public class AppConfig {
}
```
```java
public class MainApplication {

	public static void main(String[] args) {
		// 启动容器
		AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
		// 获取bean
		RegularBean bean = applicationContext.getBean(RegularBean.class);
		// 打印bean
		// 打印结果: com.tiantang.study.components.RegularBean@7a675056
		System.out.println(bean);
	}
}
```

### 1.2 通过ImportSelector接口
* Import注解可以通过ImportSelector接口的实现类来注册Bean，@Import(DemoImportRegistrar.class)。示例代码如下：。 DemoImportRegistrar是开发人员自定义的一个类，它实现了ImportSelector接口，重写了selectImports()方法，在selectImports()的返回的字符串数组中，添加了SelectorBean类的全类名，SelectorBean是自定义一个类。
```java
/**
* 自定义的一个普通类
*/
public class SelectorBean {
}
```
```java
/**
* 通过@Import导入DemoImportSelector类
* DemoImportSelector类是自定义的一个类，实现了ImportSelector接口
*/
@Configuration
@Import(DemoImportSelector.class)
public class AppConfig {
}
```
```java
/**
* 通过实现ImportSelector接口来向Spring容器中添加一个Bean
* 该类重写了ImportSelector接口的selectImports()方法
*/
public class DemoImportSelector implements ImportSelector {

	@Override
	public String[] selectImports(AnnotationMetadata importingClassMetadata) {
		// 该方法的返回值是一个String[]数组
		// 向数组中添加类的全类名，这样就能将该类注册到Spring容器中了
		return new String[]{"com.tiantang.study.components.SelectorBean"};
	}
}
```
```java
public class MainApplication {

	public static void main(String[] args) {
		// 启动容器
		AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
		// 获取bean
		SelectorBean bean = applicationContext.getBean(SelectorBean.class);
		// 打印bean
		// 打印结果:com.tiantang.study.components.SelectorBean@4ef37659
		System.out.println(bean);
	}
}
```

### 1.3 通过ImportBeanDefinitionRegistrar接口
* Import注解可以通过ImportBeanDefinitionRegistrar接口的实现类来注册Bean，@Import(DemoImportRegistrar.class)。示例代码如下：DemoImportRegistrar是开发人员自定义的一个类，它实现了ImportBeanDefinitionRegistrar接口，重写了registerBeanDefinitions()方法。在registerBeanDefinitions()中通过一段代码向Spring中注册了RegistrarBean类（RegistrarBean类是自定义的一个类）。
* 在重写的registerBeanDefinitions()方法中，加了很详细的注释，方法中的代码可能对于从未接触BeanDefinition类的API的朋友来说，可能比较陌生，多看Spring源码就，多熟悉就好了。
```java
/**
* 自定义的一个普通类
*/
public class RegistrarBean {
}
```
```java
/**
* 通过@Import导入DemoImportRegistrar类
* DemoImportRegistrar类是自定义的一个类
* 实现了ImportBeanDefinitionRegistrar接口
* 重写了registerBeanDefinitions()方法
*/
@Configuration
@Import(DemoImportRegistrar.class)
public class AppConfig {
}
```
```java
/**
* 通过实现ImportBeanDefinitionRegistrar接口，
* 重写registerBeanDefinitions()方法来向Spring容器汇总注册一个bean
*/
public class DemoImportRegistrar implements ImportBeanDefinitionRegistrar {

	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		// 通过BeanDefinitionBuilder来创建一个BeanDefinition(建造者设计模式了解一下)
		// 这里也可以直接通过关键字new来创建一个BeanDefinition。由于BeanDefinition是一个接口，接口是不能new的，因此需要new它的实现类
		// 例如: GenericBeanDefinition genericBeanDefinition = new GenericBeanDefinition();
		// 		 genericBeanDefinition.setBeanClass(RegistrarBean.class);
		// 上面两行代码完全是下面两行代码等价的。当然也可以new一个AnnotatedBeanDefinition。我们写的AppConfig类就是被Spring解析成一个AnnotatedBeanDefinition
		// 这里其实有很多API，例如BeanDefinitionRegistry中注册bean的方法，BeanDefinition中为bean设置相关特性的方法
		BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(RegistrarBean.class);
		AbstractBeanDefinition beanDefinition = beanDefinitionBuilder.getBeanDefinition();
		
		// 上面两行代码是将RegistrarBean的解析成BeanDefinition，下面则是向Spring中注册RegistrarBean类对应的BeanDefinition
		// 注意，调用registry类的registerBeanDefinition()方法时，我们为这个Bean指定了beanName。
		registry.registerBeanDefinition("demoBean",beanDefinition);
	}
}
```
```java
public class MainApplication {

	public static void main(String[] args) {
		// 启动容器
		AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
		// 获取bean
		RegistrarBean bean = applicationContext.getBean(RegistrarBean.class);
		// 打印bean
		// 打印结果:com.tiantang.study.components.RegistrarBean@2f465398
		System.out.println(bean);

		// 通过beanName来获取bean
		RegistrarBean demoBean = (RegistrarBean)applicationContext.getBean("demoBean");
		// 打印结果:com.tiantang.study.components.RegistrarBean@2f465398
		// 和上面的打印结果是一样的，这也说明两者是同一个对象
		System.out.println(demoBean);
	}
}
```

## 2. @Import的原理
> 通过上面的三个Demo，了解了Import注解的三种用法，是不是发现我们不用通过@ConponentScan和@Component等注解，我们也能向Spring容器中注册Bean？那么问题来了，为什么通过Import注解，就能实现向Spring容器中注册bean？原理是什么？

* 我们可以先猜想一下：
* 猜想一：我们都知道Spring能在启动时，自动帮我们创建好Bean，那么这个`自动`到底是怎么实现的呢？在Spring容器中，Spring容器要想为某个类创建实例对象，就必须先把对应的class类解析为BeanDefinition，然后才能实例化对象。那么我们通过Import注解的方式向容器中注册Bean，也是`一定`会先把要注册的类的class解析为BeanDefinition。
* 猜想二：如果有了猜想一，那么又会出现一个新的问题：什么时候把这些class变为BeanDefiniton的呢，如何针对Import注解做到特殊处理？Spring容器在启动阶段，有两个很重要的过程，一个是通过BeanFactoryPostProcessor后置处理器参与BeanFactory的建造，另外一个就是通过BeanPostProcessor后置处理器来参与Bean的建造。后者是创建Bean，而要创建Bean则需要BeanDefinition，所以Import注解的注解不会在这个过程。而前者是BeanFactory的建造过程，根据类名就能猜出，BeanFactory是一个Bean工厂，所以BeanDefinition作为创建Bean的原料，很有可能就是在这一步对Import注解做了特殊处理，解析出了要注册Bean的BeanDefinition。
* 在上一篇文章中（ [点击此处查看上一篇文章](https://mp.weixin.qq.com/s/f2vSH9YNmnNqdps05LEEHw)）详细介绍了Spring中一个非常重要的类：`ConfigurationClassPostProcessor`，而这个类就是BeanFactoryPostProcessor的实现类，参与了BeanFactory的建造。这个类处理了@Configuration、@ComponentScan等注解，实际上，Import注解也是在这一步被处理的。

> 接下来就看下Import注解的实现原理。在Spring容器自动过程中，会执行refresh()方法，refresh()方法中会调用postProcessBeanFactory()。在postProcessBeanFactory()方法中又会执行所有BeanFactoryPostProcessor后置处理器。那么就会执行到ConfigurationClassPostProcessor的postProcessBeanDefinitionRegistry()方法，而在该方法中调用了processConfigBeanDefinitions()方法。下面是processConfigBeanDefinitions()的部分代码(只保留了几行和今天内容有关的代码，可以阅笔者的上一篇文章[点击此处查看上一篇文章](https://mp.weixin.qq.com/s/f2vSH9YNmnNqdps05LEEHw)，里面详细介绍了该方法的全部代码)。

```java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
	
	// 创建一个parser
	ConfigurationClassParser parser = new ConfigurationClassParser(
			this.metadataReaderFactory, this.problemReporter, this.environment,
			this.resourceLoader, this.componentScanBeanNameGenerator, registry);

	do {
		// 解析配置类，在此处会解析配置类上的注解(ComponentScan扫描出的类，@Import注册的类，以及@Bean方法定义的类)
		// 注意：这一步只会将加了@Configuration注解以及通过@ComponentScan注解扫描的类才会加入到BeanDefinitionMap中
		// 通过其他注解(例如@Import、@Bean)的方式，在parse()方法这一步并不会将其解析为BeanDefinition放入到BeanDefinitionMap中，而是先解析成ConfigurationClass类
		// 真正放入到map中是在下面的this.reader.loadBeanDefinitions()方法中实现的
		parser.parse(candidates);
		
		// 将上一步parser解析出的ConfigurationClass类加载成BeanDefinition
		// 实际上经过上一步的parse()后，解析出来的bean已经放入到BeanDefinition中了，但是由于这些bean可能会引入新的bean，例如实现了ImportBeanDefinitionRegistrar或者ImportSelector接口的bean，或者bean中存在被@Bean注解的方法
		// 因此需要执行一次loadBeanDefinition()，这样就会执行ImportBeanDefinitionRegistrar或者ImportSelector接口的方法或者@Bean注释的方法
		this.reader.loadBeanDefinitions(configClasses);
		
	}
	while (!candidates.isEmpty());

}
```
* 在parser.parse()方法中，最终会调用到doProcessConfigurationClass()方法。
```java
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass) throws IOException {
	// 省略无关代码...

	// Process any @Import annotations
	// 处理Import注解注册的bean，这一步只会将import注册的bean变为ConfigurationClass,不会变成BeanDefinition
	// 而是在loadBeanDefinitions()方法中变成BeanDefinition，再放入到BeanDefinitionMap中
	processImports(configClass, sourceClass, getImports(sourceClass), true);
	
	// 省略无关代码...
	return null;
}
```
* 可以看到，processImports()方法处理了Import注解。在processImports()方法中，分别对Import注解的三种情况做了处理。方法作用的解释和源码如下:
```java
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
Collection<SourceClass> importCandidates, boolean checkForCircularImports) {

	for (SourceClass candidate : importCandidates) {
		if (candidate.isAssignable(ImportSelector.class)) {
			// 处理DeferredImportSelector的实现类，回调开发人员重写的selectImports()方法
			Class<?> candidateClass = candidate.loadClass();
			ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
			ParserStrategyUtils.invokeAwareMethods(
					selector, this.environment, this.resourceLoader, this.registry);
			if (this.deferredImportSelectors != null && selector instanceof DeferredImportSelector) {
				this.deferredImportSelectors.add(
						new DeferredImportSelectorHolder(configClass, (DeferredImportSelector) selector));
			}
			else {
				// 处理DeferredImportSelector的实现类，回调开发人员重写的selectImports()方法
				// 返回值是一个字符串数组，数组元素为类的全类名，然后把全类名变为SourceClass
				// 为什么要变为SourceClass呢？因为在此处解析时，Spring是通过SourceClass来解析类的
				String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
				Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
				// 递归调用processImports,为什么要递归调用该方法？
				// 因为上面返回的全类名所表示的类可能是ImportSelector或者ImportBeanDefinitionRegistrar
				processImports(configClass, currentSourceClass, importSourceClasses, false);
			}
		}
		else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
			// 处理ImportBeanDefinitionRegistrar类
			Class<?> candidateClass = candidate.loadClass();
			ImportBeanDefinitionRegistrar registrar =
					BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
			// 在此处回调开发人员重写的ImportBeanDefinitionRegistrar的registerBeanDefinitions()方法		
			ParserStrategyUtils.invokeAwareMethods(
					registrar, this.environment, this.resourceLoader, this.registry);
			configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
		}
		else {
			// 处理通过Import注解导入的普通类，例如本次Demo中的RegularBean
			// 这里只需要直接调用processConfigurationClass()方法即可，把RegularBean当做一个配置类去解析
			// 因为RegularBean这个类型可能加了@ConponentScan，@Bean等注解
			this.importStack.registerImport(
					currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
			processConfigurationClass(candidate.asConfigClass(configClass));
		}
	}
}
```

* 在parse()方法执行完，通过Import注解注册的类，此时还没有将对应的BeanDefinition加入工厂的BeanDefinitionMap中，而只是将class类解析为成ConfigurationClass对象了。为什么呢？笔者也没想没明白，猜测可能是因为parse()方法只是用来做解析Class用，而且解析出来的类可能又是一些特殊的配置类，例如类中含有@Bean注解，@Import注解，或者是Spring中拓展点接口的实现类。所以暂时没有将其加到BeanDefinitionMap中
* 执行完parse()方法后，接着通过loadBeanDefinitions()方法，将解析出来的ConfigurationClass类变为BeanDefinition，然后放入到BeanDefinitionMap中。在loadBeanDefinition()方法中最终会调用到registerBeanDefinitionForImportedConfigurationClass()。
* 源码如下。看到下面的代码是不是有一种很熟悉的感觉？没错，上面demo中DemoImportRegistrar类中的代码就是参考这儿写的。在实际工作中碰到类似的场景，需要我们向容器中添加一个BeanDefiniton，就可以参考这儿的示例代码去写。(^-^这也算是看源码的一个好处吧)
```java
private void registerBeanDefinitionForImportedConfigurationClass(ConfigurationClass configClass) {
	// metadata中包含的就是我们要注册的类的信息，例如本次demo中的RegularBean、SelectorBean、RegistrarBean
	AnnotationMetadata metadata = configClass.getMetadata();
	// new一个BeanDefinition的实现类
	AnnotatedGenericBeanDefinition configBeanDef = new AnnotatedGenericBeanDefinition(metadata);

	ScopeMetadata scopeMetadata = scopeMetadataResolver.resolveScopeMetadata(configBeanDef);
	configBeanDef.setScope(scopeMetadata.getScopeName());
	String configBeanName = this.importBeanNameGenerator.generateBeanName(configBeanDef, this.registry);
	AnnotationConfigUtils.processCommonDefinitionAnnotations(configBeanDef, metadata);

	BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(configBeanDef, configBeanName);
	definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
	// 通过registry对象，将BeanDefinition注册到BeanDefinitionMap中
	this.registry.registerBeanDefinition(definitionHolder.getBeanName(), definitionHolder.getBeanDefinition());
	configClass.setBeanName(configBeanName);

}
```

## 3. @Enable系列注解
> 在实际工作当中，我们经常会碰到带有@Enable前缀的注解，通常我们称之为开启某某功能，例如@EnableAsync（开启@Async注解实现异步执行的功能）、@EnableScheduling（开启@Scheduling注解实现定时任务的功能）、@EnableTransactionManagement（开启事物管理的功能）等注解，尤其是现在绝大部分项目中都要SpringBoot框架搭建，接入了SpringCloud等微服务，遇见@Enable系列的注解更是家常便饭，例如：@EnableDiscoveryClient、@EnableFeignClients。既然这么常见，那就很有必要知道@Enable系列注解的原理了。

* 下面是EnableAsync注解的源代码
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AsyncConfigurationSelector.class)
public @interface EnableAsync {

	Class<? extends Annotation> annotation() default Annotation.class;

	boolean proxyTargetClass() default false;

	AdviceMode mode() default AdviceMode.PROXY;

	int order() default Ordered.LOWEST_PRECEDENCE;

}
```

* 在开发工具中，我们选择一个@Enable的注解，例如：@EnableAsync注解，我们查看一下这个注解的源码。我们可以看到这个注解上面有加了四个注解，@Target、@Retention、@Documented这三个注解比较常见，另外一个注解就是@Import，今天这篇文章的主角。

> 简单介绍下@Target、@Retention、@Documented这三个注解的作用。@Target注解用来指明我们定义的注解可以作用在什么地方，例如ElementType.TYPE表示可以作用在类上、ElementType.METHOD表示可以作用在方法上，其他枚举值可以参考ElementType的枚举类。@Retention注解是指明我们定义的注解的生命周期，RetentionPolicy.RUNTIME表示在JVM虚拟机加载我们的class文件时，仍保留我们所写的注解；RetentionPolicy.SOURCE表示注解在Java源文件中存在，当编译成class文件时就去除了我们定义的注解信息；RetentionPolicy.CLASS表示当类编译成class文件时，我们自定义的注解信息仍存在，但当虚拟机加载class文件时，不会加载我们自定义的注解。@Documented注解表示注释会成为API文档中展示。

* 我们发现在@EnableAsync这个注解中，出现了@Import注解。在@Import注解引入了AsyncConfigurationSelector类。看类名就知道，这个类实现了ImportSelector接口。
```java
/**
* 该类继承了AdviceModeImportSelector，而AdviceModeImportSelector实现了ImportSelector接口
* 在父类中重写了selectImports(AnnotationMetadata importingClassMetadata)。
* 同时在父类中又重载了selectImports(AdviceMode adviceMode)。
*/
public class AsyncConfigurationSelector extends AdviceModeImportSelector<EnableAsync> {

	private static final String ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME =
			"org.springframework.scheduling.aspectj.AspectJAsyncConfiguration";

	@Override
	public String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			case PROXY:
				return new String[] { ProxyAsyncConfiguration.class.getName() };
			case ASPECTJ:
				return new String[] { ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME };
			default:
				return null;
		}
	}

}
```
* AsyncConfigurationSelector的父类的源代码
```java
public abstract class AdviceModeImportSelector<A extends Annotation> implements ImportSelector {

	public static final String DEFAULT_ADVICE_MODE_ATTRIBUTE_NAME = "mode";

	protected String getAdviceModeAttributeName() {
		return DEFAULT_ADVICE_MODE_ATTRIBUTE_NAME;
	}

	// 重写ImportSelector接口中的方法
	@Override
	public final String[] selectImports(AnnotationMetadata importingClassMetadata) {
		Class<?> annoType = GenericTypeResolver.resolveTypeArgument(getClass(), AdviceModeImportSelector.class);
		AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(importingClassMetadata, annoType);
		if (attributes == null) {
			throw new IllegalArgumentException(String.format(
				"@%s is not present on importing class '%s' as expected",
				annoType.getSimpleName(), importingClassMetadata.getClassName()));
		}
		// @EnableAsync注解中有个mode属性，可以指定一个值，此处是获取指定的值。
		// 开发人员在使用@EnableAsync如果没有指定具体值，则使用@EnableAsync注解中的默认值AdviceMode.PROXY
		AdviceMode adviceMode = attributes.getEnum(this.getAdviceModeAttributeName());
		// 调用子类的selectImports()方法
		String[] imports = selectImports(adviceMode);
		if (imports == null) {
			throw new IllegalArgumentException(String.format("Unknown AdviceMode: '%s'", adviceMode));
		}
		return imports;
	}

	// 重载selectImports()方法
	protected abstract String[] selectImports(AdviceMode adviceMode);

}
```

* 从源码中可以知道，最终会调用到AsyncConfigurationSelector类的selectImports()方法。该方法中根据EnableAsync注解中指定的mode属性的值，来返回不同的全类名，从而向Spring容器中注册不同类型的Bean。在此处就是根据指定的代理类型，来向Spring容器中注册ProxyAsyncConfiguration类或者AspectJAsyncConfiguration。前者是通过JDK的动态代理方式来增强加了@Async注解的方法或者类，后者是通过AspectJ的方式来增强目标方法或者类。
> @Async的作用就是让方法异步执行，所以需要对目标方法进行增强，那么可以采用代理的方式或者AspectJ技术操作字节码，对字节码进行静态织入，从而达到目标。

* 同理，再去看看@EnableScheduling、@EnableTransactionManagement、@EnableDiscoveryClient、@EnableFeignClients等注解，是不是在它们的源码里面，都有一个@Import，在Import注解中添加的类要么是框架或者开发自定义的类，要么是ImportSelector接口的实现类，或者是ImportBeanDefinitionRegistrar接口的实现类。而在它们各自的实现类中，重写了接口的方法，向Spring容器中添加了一个带有特殊功能的类，从而达到开启某某功能的目的。
* 在SpringBoot中，通常会需要整合第三方jar包，通常我们的做法是先引入一个starter，然后在配置类或者启动类上加一个@EnableXXX，这样就和第三方jar包整合完毕了。阅读完本文，现在应该都知道这个原理了吧。

## 4. 为什么要用？
> 通过上面分析我们知道，@Enable系列注解，就是向容器中注册一个Bean，既然注册一个Bean，我们为什么不通过@Component等注解注册呢？而要用@Import注解这种方式？

* 答案很明显，灵活。例如第三方jar包，可能在某些项目中并不需要使用该jar包中的某些功能，如果我们直接在类的代码上加上@Component注解，这样在和Spring整合时，首先要确保Spring的ComponentScan注解能扫描到第三方jar包中类所在的包，其次，这样Spring容器启动后，不管用不用这个功能，都会在容器中添加这个Bean，这样就不太合适，所以灵活度不够。而用@EnableXXX注解，就能达到想用就用，不想用就关闭的目的，而且还不需要确保Spring扫描到这个第三方jar包的包名。


## 5. 总结
* 本文先通过3个demo介绍了Import注解的3种使用场景，然后结合ConfigurationClassPostProcessor类的源码分析了Import注解的使用原理。
* 接着通过@Import注解，揭开了@Enable系列注解的神秘面纱。并结合@EnableAsync注解的源码，举例说明了@Enable注解的原理。
* 最后解释了使用@Import和@Enable系列注解的好处。
* 看到这儿，是不是可以立马自己去写一个@Enable注解的组件了呢？或者自己写一个第三方的插件包了呢？

## 6. 推荐

> 最后推荐一款本人所在公司开源的性能监控工具——`Pepper-Metrics`

* 地址： https://github.com/zrbcool/pepper-metrics
* 或者 [点击此处跳转](https://github.com/zrbcool/pepper-metrics)
* `Pepper-Metrics`是坐我对面的两位同事一起开发的开源组件，主要功能是通过比较轻量的方式与常用开源组件（`jedis/mybatis/motan/dubbo/servlet`）集成，收集并计算`metrics`，并支持输出到日志及转换成多种时序数据库兼容数据格式，配套的`grafana dashboard`友好的进行展示。项目当中原理文档齐全，且全部基于`SPI`设计的可扩展式架构，方便的开发新插件。另有一个基于`docker-compose`的独立`demo`项目可以快速启动一套`demo`示例查看效果`https://github.com/zrbcool/pepper-metrics-demo`。如果大家觉得有用的话，麻烦给个`star`，也欢迎大家参与开发，谢谢：）

> 扫描下方二维码即可关注微信公众号`菜鸟飞呀飞`，一起阅读更多Spring源码。
