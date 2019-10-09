> 点击上方`菜鸟飞呀飞`，即可关注微信公众号。

[TOC]

## 1. 问题
> 在平时工作中，只要是做Java开发，基本都离不开Spring框架，Spring的一大核心功能就是IOC，它能帮助我们实现自动装配，基本上每天我们都会使用到@Autowired注解来为我们自动装配属性，那么你知道Autowired注解的原理吗？在阅读本文之前，可以先思考一下以下几个问题。

* @Autowired注解是如何实现自动装配的？
* 当为类型为A的Bean装配类型为B的属性时，如果此时Spring容器中存在多个类型为B的bean，此时Spring是如何处理的？
* 自动装配的模型是什么？有哪几种？和Autowired注解有什么关联？

## 2. Demo
> 先按照如下示例搭建一下本文的demo工程

* pom依赖
```java
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>5.1.8.RELEASE</version>
</dependency>
```

* 配置类，在配置类中指定扫描哪些包下的文件
```java
@Configuration
@ComponentScan("com.tiantang.study")
public class AppConfig {
}
```

* 定义两个service接口以及实现类，然后在OrderServiceImpl中为其注入UserService的实现类
```java
public interface UserService {
}
```
```java
@Service
public class UserServiceImpl implements UserService {
}
```
```java
public interface OrderService {

    void query();

}
```
```java
@Service
public class OrderServiceImpl implements OrderService {

    @Autowired
    private UserService userService;

    public void query(){
        System.out.println(userService);
    }
}
```

* 启动类，在启动类中通过调用getBean()方法获取到OrderService类，然后调用query()方法，在query()方法中会打印注入的UserService对象
```java
public class MainApplication {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        OrderService orderService = applicationContext.getBean(OrderService.class);
        orderService.query();
    }
}
```
* 运行main()方法，最终会在query()方法中打印出UserService对象。

## 3. 实现原理：AutowiredAnnotationBeanPostProcessor
> 通过上面的demo我们完成了对OrderServiceImpl的自动装配，为其属性userService完成了赋值操作，那么Spring是如何通过@Autowired来实现赋值的呢？我们知道，Spring在容器启动阶段，会先实例化bean，然后再对bean进行初始化操作。在初始化阶段，会通过调用Bean后置处理来完成对属性的赋值等操作，那么同理，要想实现@Autowired的功能，肯定也是通过后置处理器来完成的。这个后置处理器就是AutowiredAnnotationBeanPostProcessor。接下来我们就来看看这个类的源码。

### 3.1 何时被加入
* 在分析AutowiredAnnotationBeanPostProcessor的工作原理之前，我们需要先知道它是何时被加入到Spring容器当中的。毕竟只有先将它放入到容器中了，才能让它工作。
* 当我们调用`AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class)`启动容器的时候，在构造方法中会调用到`this()`方法，在`this()`方法中最终会调用到`registerAnnotationConfigProcessors()`方法，在该方法中，Spring会向容器注册7个Spring内置的Bean，其中就包括AutowiredAnnotationBeanPostProcessor。如果想详细了解Spring容器的启动过程，可以参考笔者的另一篇文章：[Spring源码系列之容器启动流程](https://mp.weixin.qq.com/s/q6zs7xRjpcB4YxLw6w477w)
```java
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
            BeanDefinitionRegistry registry, @Nullable Object source) {

    // 省略部分代码...
    
    Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);
    // 注册AutowiredAnnotationBeanPostProcessor,这个bean的后置处理器用来处理@Autowired的注入
    if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // 注册CommonAnnotationBeanPostProcessor，用来处理如@Resource等符合JSR-250规范的注解
    if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
    }
    return beanDefs;
}
```

### 3.2 何时被调用
> AutowiredAnnotationBeanPostProcessor是何时被调用的呢？

* Spring在创建bean的过程中，最终会调用到doCreateBean()方法，在doCreateBean()方法中会调用populateBean()方法，来为bean进行属性填充，完成自动装配等工作。populateBean()方法的部分源码如下。

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    if (hasInstAwareBpps || needsDepCheck) {
        if (pvs == null) {
            pvs = mbd.getPropertyValues();
        }
        PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        if (hasInstAwareBpps) {
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    // 执行后置处理器，填充属性，完成自动装配
                    // 在这里会调用到AutowiredAnnotationBeanPostProcessor
                    pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                    if (pvs == null) {
                        return;
                    }
                }
            }
        }
    }
}
```
* 在populateBean()方法中一共调用了两次后置处理器，第一次是为了判断是否需要属性填充，如果不需要进行属性填充，那么就会直接进行return，如果需要进行属性填充，那么方法就会继续向下执行，后面会进行第二次后置处理器的调用，这个时候，就会调用到AutowiredAnnotationBeanPostProcessor的postProcessPropertyValues()方法，在该方法中就会进行@Autowired注解的解析，然后实现自动装配。（注意：上面的源码中，只贴出了执行一次后置处理器的代码，另外一次被省略了）。
* postProcessorPropertyValues()方法的源码如下，在该方法中，会先调用findAutowiringMetadata()方法解析出bean中带有@Autowired注解、@Inject和@Value注解的属性和方法。然后调用metadata.inject()方法，进行属性填充。
```java
public PropertyValues postProcessPropertyValues(
            PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeanCreationException {
    // 解析出bean中带有@Autowired注解、@Inject和@Value注解的属性和方法
    // 对于本文的demo而言，在此处就会解析出OrderServiceImpl类上的userService属性
    // 至于如何解析的，findAutowiringMetadata()方法比较复杂，这里就不展开了，Spring中提供了很多对注解等元数据信息读取的方法，进行了大量的封装。
    // 如果不是自己亲自参与开发Spring的话，很难摸透它封装的那些数据结构。
    InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
    try {
        // 自动装配，实现依赖注入
        metadata.inject(bean, beanName, pvs);
    }
    catch (BeanCreationException ex) {
        throw ex;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
    }
    return pvs;
}
```
#### 3.2.1 findAutowiringMetadata()
* 为什么上面分析时，只说了findAutowiringMetadata()方法会解析出Autowired注解、@Inject和@Value注解呢？这是因为在AutowiredAnnotationBeanPostProcessor中定义了这样一个全局变量：
```java
private final Set<Class<? extends Annotation>> autowiredAnnotationTypes = new LinkedHashSet<>(4);
```
* 这个全局变量在AutowiredAnnotationBeanPostProcessor的构造方法中进行了初始化。初始化时，只向这个set集合中添加了三个元素：@Autowired、@Inject、@Value。其中@Inject注解是JSR-330规范中的注解。当调用findAutowiringMetadata()方法时，会根据autowiredAnnotationTypes这个全局变量中的元素类型来进行注解的解析，因此上面只说了findAutowiringMetadata()方法会解析出Autowired注解、@Inject和@Value注解这三个注解。
```java
public AutowiredAnnotationBeanPostProcessor() {
    // 添加@Autowired
    this.autowiredAnnotationTypes.add(Autowired.class);
    // 添加@Value
    this.autowiredAnnotationTypes.add(Value.class);
    try {
        // 如果项目中引入了JSR-330相关的jar包，那么就会添加@Inject
        this.autowiredAnnotationTypes.add((Class<? extends Annotation>)
                ClassUtils.forName("javax.inject.Inject", AutowiredAnnotationBeanPostProcessor.class.getClassLoader()));
    }
    catch (ClassNotFoundException ex) {
        
    }
}
```

#### 3.2.2 metadata.inject()
* 对于属性上加了Autowired注解的，经过上一步解析后，会将字段解析为AutowiredFieldElement类型；如果是方法上加了Autowired注解，则会解析为AutowiredMethodElement类型。它们均是InjectedElement类的子类，里面封装了属性名，属性类型等信息。对于@Resource,@LookUp等注解被解析后，也会解析成对应的InjectedElement的子类：ResourceElement、LookUpElement，但是这两个注解是在其他后置处理器中被解析出来的，并不是在AutowiredAnnotationBeanPostProcessor中解析的。
* metadata.inject()方法最终会调用InjectedElement类的inject()方法。对于本文中的demo，此时userService属性被解析后对应的InjectedElement是AutowiredFieldElement类，所以在此时会调用AutowiredFieldElement.inject()方法。
* 下面是AutowiredFieldElement.inject()方法的源码。在源码中加了部分注释，从源码中可以发现，核心代码是`beanFactory.resolveDependency()`。
```java
protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
        Field field = (Field) this.member;
        Object value;
        // 判断缓存（第一次注入userService的时候，肯定没有缓存，所以会进入到else里面）
        // 当第一次注入完成后，会将userService缓存到cachedFieldValue这个属性中，
        // 这样当其他的类同样需要注入userService时，就会从这儿的缓存当中读取了。
        if (this.cached) {
            value = resolvedCachedArgument(beanName, this.cachedFieldValue);
        }
        else {
            DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
            desc.setContainingClass(bean.getClass());
            Set<String> autowiredBeanNames = new LinkedHashSet<>(1);
            TypeConverter typeConverter = beanFactory.getTypeConverter();
            try {
                // 通过beanFactory.resolveDependency()方法，来从容器中找到userService属性对应的值。
                value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
            }
            catch (BeansException ex) {
                throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(field), ex);
            }
            // 省略部分代码...
            // 省略的这部分代码就是将value进行缓存，缓存到cachedFieldValue属性中
        }
        if (value != null) {
            // 通过Java的反射，为属性进行复制
            ReflectionUtils.makeAccessible(field);
            field.set(bean, value);
        }
    }
}
```
* 在beanFactory.resolveDependency()方法中主要调用了doResolveDependency()方法，下面我们重点分析一下doResolveDependency()这个方法，这个方法的代码很长。源码如下：
```java
public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
            @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

    InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
    try {
        Object shortcut = descriptor.resolveShortcut(this);
        if (shortcut != null) {
            return shortcut;
        }
        // 省略部分不重要的代码...
        
        // 属性的类型可能是数组、集合、Map类型，所以这一步是处理数组类型、Collection、Map类型的属性
        Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
        if (multipleBeans != null) {
            return multipleBeans;
        }
        // 根据需要注入的类型type，从容器中找到有哪些匹配的Bean。
        Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
        // 如果从容器中没有找到，且@Autowired的required属性为true，那么则会抛出异常
        if (matchingBeans.isEmpty()) {
            if (isRequired(descriptor)) {
                raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
            }
            return null;
        }

        String autowiredBeanName;
        Object instanceCandidate;

        // 先根据类型匹配出可以依赖注入的bean的Class，如果匹配出多个，则再根据属性名匹配
        if (matchingBeans.size() > 1) {
            autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
            if (autowiredBeanName == null) {
                if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
                    // 当匹配到多个bean的Class，但是却不知道要选择哪一个注入时，就会抛出异常
                    return descriptor.resolveNotUnique(type, matchingBeans);
                }
                else {
                    // In case of an optional Collection/Map, silently ignore a non-unique case:
                    // possibly it was meant to be an empty collection of multiple regular beans
                    // (before 4.3 in particular when we didn't even look for collection beans).
                    return null;
                }
            }
            instanceCandidate = matchingBeans.get(autowiredBeanName);
        }
        else {
            // We have exactly one match.
            // 只匹配到一个，则就使用匹配到这个类型
            Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
            autowiredBeanName = entry.getKey();
            instanceCandidate = entry.getValue();
        }

        if (autowiredBeanNames != null) {
            autowiredBeanNames.add(autowiredBeanName);
        }
        // 此处instanceCandidate = UserServiceImpl.class
        if (instanceCandidate instanceof Class) {
            // instanceCandidate是注入属性的类型，这个需要根据Class，通过FactoryBean的getBean()方法，创建该类型的单例bean
            instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
        }
        Object result = instanceCandidate;
        if (result instanceof NullBean) {
            // 如果没从容器中找到对应的bean，则需要判断属性值是否是必须注入的，
            // 即@Autowired(required=false/true),如果为true,则抛异常，这就是经常项目启动时，我们看到的异常
            if (isRequired(descriptor)) {
                raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
            }
            result = null;
        }
        if (!ClassUtils.isAssignableValue(type, result)) {
            throw new BeanNotOfRequiredTypeException(autowiredBeanName, type, instanceCandidate.getClass());
        }
        return result;
    }
    finally {
        ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
    }
}
```
* 在上面的源码中，我们看到调用了resolveMultipleBeans()，这个方法是为了处理属性时数组、集合、Map的情况的，例如我们在OrderServiceImpl中进行如下注入：
```java
@Autowired
private UserService[] userServiceArray;

@Autowired
private List<UserService> userServiceList;

@Autowired
private Map<String,UserService> userServiceMap;
```
* 这个时候，resolveMultipleBeans()方法就会从容器中找到所有的UserService类型的Bean。resolveMultipleBeans()方法中会调用findAutowireCandidate()方法，从容器中找到对应类型的bean，实际上最终会调用getBean()方法，这样当UserService类型的对象还没被创建时，就会去创建。（Spring中无论是创建Bean还是获取Bean，入口都是getBean()方法，如果需要详细了解该方法，可以阅读比这的另一篇文章：[通过源码看Bean的创建过程](https://mp.weixin.qq.com/s/WwjicbYtcjRNDgj2bRuOoQ)）。
* resolveMultipleBeans()方法的部分源码
```java
private Object resolveMultipleBeans(DependencyDescriptor descriptor, @Nullable String beanName,
            @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) {
    Class<?> type = descriptor.getDependencyType();
    // 处理数组类型
    if (type.isArray()) {
        // findAutowireCandidates最终会调用getBean()方法
        Map<String, Object> matchingBeans = findAutowireCandidates(beanName, componentType,
                new MultiElementDescriptor(descriptor));
        return result;
    }
    // 处理集合类型
    else if (Collection.class.isAssignableFrom(type) && type.isInterface()) {
        // findAutowireCandidates最终会调用getBean()方法
        Map<String, Object> matchingBeans = findAutowireCandidates(beanName, elementType,
                new MultiElementDescriptor(descriptor));
        return result;
    }
    // 处理Map类型
    else if (Map.class == type) {
        // findAutowireCandidates最终会调用getBean()方法
        Map<String, Object> matchingBeans = findAutowireCandidates(beanName, valueType,
                new MultiElementDescriptor(descriptor));
        return matchingBeans;
    }
    else {
        return null;
    }
}
```
* 如果需要注入的属性是普通类型(非数组、集合、Map)，那么方法会继续向下执行，会调用下面一行代码，根据属性的类型来查找bean。
```java
Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
```
* 可以看到又是调用findAutowireCandidates()方法，该方法最终会调用getBean()方法，所以它会从容器中找到对应类型的bean，即UserService类型的Bean。如果容器没有找到对应类型的Bean，且属性是必须注入的，即Autowired注解的required属性为true，那么就会抛出异常，这个异常就是我们经常看见的：`NoSuchBeanDefinitionException`。
```java
// 如果容器汇总没有找到指定类型的bean，那么matchingBeans属性就是空的
if (matchingBeans.isEmpty()) {
    if (isRequired(descriptor)) {
        // 抛出异常
        raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
    }
    return null;
}
```
* matchingBeans这个map的大小可能大于1，因为从容器中找到了多个满足类型的Bean，即可能找到多个UserService类型的bean，那么这个时候就需要判断这多个bean中，究竟应该注入哪一个。
```java
if (matchingBeans.size() > 1) {
    // 判断应该使用哪一个bean
    autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
    if (autowiredBeanName == null) {
        if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
            // 当匹配到多个bean的Class，但是却不知道要选择哪一个注入时，就会抛出异常
            return descriptor.resolveNotUnique(type, matchingBeans);
        }
        else {
            return null;
        }
    }
    instanceCandidate = matchingBeans.get(autowiredBeanName);
}
```
* 在上面的代码中，会调用determineAutowireCandidate()方法，判断应该使用哪一个bean。如果determineAutowireCandidate()方法也无法决定使用哪一个，determineAutowireCandidate()就会返回null。这个时候如果属性又是必须注入的，即@Autowired的required=true，那么就会抛出异常（因为required=true表示这个属性是必须注入的，但是程序又不知道需要注入哪一个，所以就会出错），这个异常也是我们经常见到的：`NoUniqueBeanDefinitionException`。
```java
"expected single matching bean but found " + beanNamesFound.size() + ": " + StringUtils.collectionToCommaDelimitedString(beanNamesFound)
```
* 那么determineAutowireCandidate()又是如何判断应该决定使用哪一个bean的呢？Spring会先找到加了@Primary注解的bean，比如Spring找到了两个UserService类型的Bean，分别为UserServiceImpl1和UserServiceImpl2，如果UserServiceImpl1的类上加了@Primary注解，那么就会优先使用userServiceImpl1类型的Bean进行注入。如果都没有加@Primary注解，那么就会找加了@Priority注解的bean，优先级最高的会被优先选中。如果都没有加@Priority，那么就会根据属性的名称和Spring中beanName来判断，例如：UserServiceImpl1和UserServiceImpl2在Spring容器中beanName分别是userServiceImpl1和userServiceImpl2。如果此时OrderServiceImpl中需要注入的UserService类型的属性名是userServiceImpl1，那么Spring就会为其注入UserServiceImpl1类型的单例对象；如果属性名是userServiceImpl2，那么Spring就会为其注入UserServiceImpl2。如果属性名与userServiceImpl1不相等，也与userServiceImpl2不相等，那么就会返回null，那么在方法的调用处，就会抛出`NoUniqueBeanDefinitionException`异常。
* `这就是我们平常所说的@Autowired注解是先根据类型注入，当碰到多个相同类型时，就会根据属性名注入。它的实现原理就是在如下代码中实现的。`
```java
protected String determineAutowireCandidate(Map<String, Object> candidates, DependencyDescriptor descriptor) {
    Class<?> requiredType = descriptor.getDependencyType();
    // 根据Primary注解来决定优先注入哪个bean
    String primaryCandidate = determinePrimaryCandidate(candidates, requiredType);
    if (primaryCandidate != null) {
        return primaryCandidate;
    }
    // 根据@Priority注解的优先级来决定注入哪个bean
    String priorityCandidate = determineHighestPriorityCandidate(candidates, requiredType);
    if (priorityCandidate != null) {
        return priorityCandidate;
    }
    // 如果既没有指定@Primary，也没有指定@Priority，那么就会根据属性的名称来决定注入哪个bean
    // 如果要注入的属性的名称与Bean的beanName相同或者别名相同，那么会就会优先注入这个Bean
    for (Map.Entry<String, Object> entry : candidates.entrySet()) {
        String candidateName = entry.getKey();
        Object beanInstance = entry.getValue();
        if ((beanInstance != null && this.resolvableDependencies.containsValue(beanInstance)) ||
                matchesBeanName(candidateName, descriptor.getDependencyName())) {
            return candidateName;
        }
    }
    // 前面三种情况都没有确定要注入哪个bean，那么就返回null。当返回null时，那么就会再调用该方法出抛出异常。
    return null;
}
```
* 当matchingBeans大于1，且通过determineAutowireCandidate()方法确认了使用哪个bean注入时，或者当matchingBeans=1时，后面就会根据确定的beanName，来从容器中找到对应的Bean，然后将Bean返回，最后在`AutowiredFeildElement.inject()`方法中，通过反射进行注入，完成自动装配。
* 至此，AutowiredAnnotationBeanPostProcessor就通过postProcessPropertyValues()方法完成了自动装配。以上就是@Autowired注解的实现原理。

## 4. 自动装配的模型
> 在文章的开头，我问了一个问题：自动装配的模型是什么？有哪几种？和Autowired注解有什么关联？实际上这个问题，和今天的主角AutowiredAnnotationBeanPostProcessor没有任何关系。但是为什么又把它放在这篇文章中提出来呢？这是因为@Autowired注解的实现原理和自动装配模型极为容易混淆。

* 在@Autowired的实现原理中，我们提到了它会现根据类型来匹配，即byType，当类型匹配到多个时，会根据属性名匹配，即byName。而自动装配的模型有3种，即对应Autowire枚举类中的枚举类型值，枚举类中的名字也是ByName，ByType，那么这两个地方的byName和byType的含义是一样的吗。(这里说3种，实际上不是特别准确，因为从下面源码中可以看到，Autowire枚举的值实际上对应AutowireCapableBeanFactory类中的常量值，而在AutowireCapableBeanFactory中常量值有5个：AUTOWIRE_NO、AUTOWIRE_BY_NAME、AUTOWIRE_BY_TYPE、AUTOWIRE_CONSTRUCTOR、AUTOWIRE_AUTODETECT，其中AUTOWIRE_AUTODETECT目前已经被舍弃了)。
```java
public enum Autowire {

    NO(AutowireCapableBeanFactory.AUTOWIRE_NO),

    BY_NAME(AutowireCapableBeanFactory.AUTOWIRE_BY_NAME),

    BY_TYPE(AutowireCapableBeanFactory.AUTOWIRE_BY_TYPE);
}
```
* 首先可以很明确的说，这两者的含义是完全不一样的。@Autowired注解中byName和byType可以理解为实现自动装配的技术，是一种实现手段，是我们开发自己理解定义的，在Spring中，Spring并没有为这种实现手段来命名。而Autowire枚举中的值，则是Spring定义的，它是用来描述Bean的装配模型的，是用来给BeanDefinition的autowireMode属性来赋值的。Spring中默认BeanDefinition的autowireMode的值为Autowire.NO。
* 第二，Bean的注入模型可以手动指定，而@Autowired的实现原理个人是没法做出修改的。例如通过XML配置bean时进行指定，如下示例：
```java
<bean id="orderService" class="com.tiantang.study.service.impl.OrderServiceImpl" autowire="byName">
</bean>
```
* 或者通过@Bean注解的属性值指定
```java
@Bean(autowire = Autowire.BY_NAME)
public OrderService orderService(){
    return new OrderServiceImpl();
}
```
* 当我们为Bean指定了具体的装配模型时，那么这个Bean对应的BeanDefinition的autowiredMode属性值将会是我们指定的值，而不再是Spring默认的Autowire.NO。
* 那么指定装配模型有什么用处呢？它会在populateBean()方法中发挥用处。前面我们贴出的populateBean()方法的代码中，省略了一部分很重要的代码，现在我们再贴出populateBean()相对完整的代码：
```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    // 判断bean的注入模型是byName，还是byType。
    if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME || mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
        if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }
        // 对于MyBatis而言，Mapper在实例化之后，会填充属性，这个时候，需要找到MapperFactoryBean有哪些属性需要填充
        // 在Mapper的BeanDefinition初始化时，默认添加了一个属性，addToConfig
        // 在下面的if逻辑中，执行完autowireByType()方法后，会找出另外另个需要填充的属性，分别是sqlSessionFactory和sqlSessionTemplate
        if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }
        pvs = newPvs;
    }

    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

    if (hasInstAwareBpps || needsDepCheck) {
        if (pvs == null) {
            pvs = mbd.getPropertyValues();
        }
        PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        if (hasInstAwareBpps) {
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    // 第六次执行后置处理器，填充属性，完成自动装配
                    pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                    if (pvs == null) {
                        return;
                    }
                }
            }
        }
        if (needsDepCheck) {
            checkDependencies(beanName, mbd, filteredPds, pvs);
        }
    }

    if (pvs != null) {
        // 实现通过byName或者byType类型的属性注入
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}
```
* 在上面的源码中，我们发现，在调用后置处理器自动装配属性之前，会先进行判断bean的装配模型是哪一种，如果是AUTOWIRE_BY_NAME，那么就会调用autowireByName()方法，在autowireByName()方法中，会先通过setter方法找到属性，然后根据属性名从容器中查找Bean，为其属性赋值；如果是AUTOWIRE_BY_TYPE，那么就会调用autowireByType()方法，autowireByType()这个方法也是先根据setter方法查找属性，然后再根据属性的类型从容器中查找bean，为其属性赋值。
* 注意上面提到了通过setter方法查找属性，在查找时Spring会忽略调很多setter方法，例如属性的类型是基本数据类型，包装类型，Date，String,Class，Local等等，这些都会被Spring忽略。
```java
// 判断bean的注入模型是byName，还是byType。
if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME || mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
    MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
    if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME) {
        autowireByName(beanName, mbd, bw, newPvs);
    }
    // 对于MyBatis而言，Mapper在实例化之后，会填充属性，这个时候，需要找到MapperFactoryBean有哪些属性需要填充
    // 在Mapper的BeanDefinition初始化时，默认添加了一个属性，addToConfig
    // 在下面的if逻辑中，执行完autowireByType()方法后，会找出另外另个需要填充的属性，分别是sqlSessionFactory和sqlSessionTemplate
    if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
        autowireByType(beanName, mbd, bw, newPvs);
    }
    pvs = newPvs;
}
```
* Spring通过autowireByName()或者autowireByType()查到属性和属性对应的Bean值以后，会将属性和属性值放到pvs这个局部变量中，然后在populateBean()方法的最后一行，调用了`applyPropertyValues()`方法，然后将其填充到Bean当中。
* 从这儿也可以发现，注入模型和Autowired注解的自动装配技术没有任何关联关系。@Autowired需要借助AutowiredAnnotationBeanPostProcessor类来实现装配，而自动装配模型则是定义bean的装配类型是什么，然后根据装配类型来执行不同的方法，并不依赖于AutowiredAnnotationBeanPostProcessor类。
* 对于自动装配模型Autowire.BY_TYPE有一个经典的应用场景，那就是Spring和MyBatis的整合。在Spring和MyBatis整合时，每一个Mapper最后对应一个MapperFactroyBean，而MapperFactroyBean的自动装配模型就是AUTOWIRE_BY_TYPE。MapperFactroyBean在装配时，会执行到autowireByType()方法，会根据setter方法找到符合条件的两个属性`sqlSessionFactory`、`sqlSessionTemplate`，然后为这两个属性赋值。
* Spring整合MyBatis时，创建MapperFactoryBean的BeanDefinition的部分源代码如下：
```java
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
    GenericBeanDefinition definition;
    for (BeanDefinitionHolder holder : beanDefinitions) {
        definition = (GenericBeanDefinition) holder.getBeanDefinition();
        // 指定MapperFactoryBean的自动装配模型为AUTOWIRE_BY_TYPE      
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
    }
}
```
* 综上，自动装配的模型中的ByName、ByType与@Autowired装配的ByName、ByType不是同一个含义，两者完全没有关系。

## 5. 总结
> 总结之前，先回答一下文章开头的三个问题。1）@Autowired注解的实现是通过后置处理器AutowiredAnnotationBeanPostProcessor类的postProcessPropertyValues()方法实现的。2）当自动装配时，从容器中如果发现有多个同类型的属性时，@Autowired注解会先根据类型判断，然后根据@Primary、@Priority注解判断，最后根据属性名与beanName是否相等来判断，如果还是不能决定注入哪一个bean时，就会抛出NoUniqueBeanDefinitionException异常。3）@Autowired自动装配中byName、byType与自动装配的模型中的byName、byTYpe没有任何关系，两者含义完全不一样，前者是实现技术的手段，后者是用来定义BeanDefiniton中autowireMode属性的值的类型。

* 本文通过源码主要分析了@Autowired注解的实现原理是通过AutowiredAnnotationBeanPostProcessor这个后置处理器来实现的，然后针对@Autowired注入过程中如果碰到多个同类型的bean时该如何处理的实现细节做了具体的分析。
* 通过本文我们不仅知道了@Autowired注解的实现原理，还知道了@Autowired注解还能为我们注入Map、数组、Collection类型的属性。
* 接着本文通过Spring中自动装配的模型，比较了自动装配模型与@Autowired的关联关系，并通过举例Spring与MyBatis整合的例子，介绍了自动装配模型BY_TYPE的使用场景，并做了简单分析。
* 最后，看完本文，你能猜到@Resource注解的实现原理吗？如果还不清楚的话，可以看下CommonAnnotationBeanPostProcessor类的源码。
