> 点击上方`菜鸟飞呀飞`，即可关注微信公众号。

[TOC]

## 1. 前言
> 在笔者的上一篇文章中([点击此处跳转查看](https://mp.weixin.qq.com/s/y_2Z9m0gevp-cMkEIflrwA))介绍了@Import注解的使用场景和原理，以及@EnableXXX注解的实现原理，这一篇文章将通过一个自定义的@Enable注解来实现一个Redis和Spring整合的插件。

* 以前我们在Spring中整合Redis时（非SpringBoot和redis整合，看完本文，`spring-boot-starter-data-redis`这个包的原理也基本明白了），通常第一步是先引入redis客户端的jar包，第二步通过XML方式或者@Bean方式配置一个Jedis或者JedisCluster对象，第三步就是在代码中注入Jedis或者JedisCluter。这三步中，最为麻烦的是第二步。那么有没有一种方法能像@EnableXXX那样满足我们的需求呢？

## 2. 思路
> 要想完成Spring和redis的整合，我们就需要向Spring容器中添加一个Jedis或者JedisCluster这样的bean，然后还需要为redis的客户端设置属性，如连接地址，端口号等。那么我们可以自定义一个@EnableJedisClient或者@EnableJedisClusterClient注解，让这两个注解来达到我们的目的。

* 自定义@EnableJedisClient和@EnableJedisClusterClient注解，让这两个注解来向容器中注册Jedis和JedisCluster。
* 由于@Enable注解通常是结合@Import注解使用的，而@Import注解能帮助我们向容器中注册Bean，但是没办法为Bean的属性赋值，因为Import注解的处理只能干预BeanFactory的建造过程，不能参与Bean的创建过程，例如不能参与为Bean的属性赋值等操作。
* 既然想要参与Bean的创建过程，为Bean的属性赋值，那么我们可以通过BeanPostProcessor来参与Bean的创建过程，创建JedisClientBeanPostProcessor和JedisClusterClientBeanPostProcessor类分别为Jedis和JedisCluster来设置属性，这两个类均实现了BeanPostProcessor接口。

## 3. 代码实现

> 先实现Jedis的整合，再实现JedisCluster的整合。前者是针对单机版的redis，后者是针对集群版的redis。

* pom依赖
```java
 <dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>5.1.8.RELEASE</version>
    <!-- 如果该项目是准备作为一个第三方插件的话，这里对spring的依赖范围最好指定为provided-->
    <!--<scope>provided</scope>-->
</dependency>
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>3.1.0</version>
</dependency>
```

### 3.1 @EnableJedisClient

* 首先定义一个EnableJedisClient注解，在注解中通过Import注解导入了JedisClientImportRegistrar类。并且为EnableJedisClient添加了一个属性：namespace，添加该属性的目的是为了让项目中能同时引入多个redis。例如：在项目中需要同时连接两个不同的redis机器，那么这个时候就可以通过namespace来区分。
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(JedisClientImportRegistrar.class)
public @interface EnableJedisClient {

    String namespace() default "default";
}
```
* JedisClientImportRegistrar类实现了ImportBeanDefinitionRegistrar接口，重写了registerBeanDefinitions()方法，在方法中向Spring容器中注册了两个Bean，一个是Jedis，一个是JedisClientBeanPostProcessor后置处理器，注册该后置处理器是为了在后面Jedis初始化的过程中，为jedis设置连接地址，端口号等属性。
* Jedis在容器中的beanName是 namespace + "Jedis"，namespace的值是从EnableJedisClient注解中获取到的。例如如下示例使用:那么此时的namespce的值为demo，如果不指定，则为default。
```java
@EnableJedisClient(namespace = "demo")
public class AppConfig {
}
```

* 源码如下

```java
public class JedisClientImportRegistrar implements ImportBeanDefinitionRegistrar {

    public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry beanDefinitionRegistry) {
        Map<String, Object> annotationAttributes = annotationMetadata.getAnnotationAttributes(EnableJedisClient.class.getName());
        AnnotationAttributes attributes = AnnotationAttributes.fromMap(annotationAttributes);
        String namespace = attributes.getString("namespace");
        // 创建jedis的BeanDefinition,然后注册进容器中，beanName为namespace + "Jedis"
        BeanDefinitionBuilder jedisBeanDefinitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(Jedis.class);
        AbstractBeanDefinition jedisBeanDefinition = jedisBeanDefinitionBuilder.getBeanDefinition();
        beanDefinitionRegistry.registerBeanDefinition(namespace+Jedis.class.getSimpleName(),jedisBeanDefinition);

        // 向容器注册一个Jedis的后置处理器,这是为了让后置处理器为Jedis的属性赋值
        BeanDefinitionBuilder beanDefinitionBuilder = BeanDefinitionBuilder.genericBeanDefinition(JedisClientBeanPostProcessor.class);
        beanDefinitionRegistry.registerBeanDefinition(JedisClientBeanPostProcessor.class.getSimpleName(),beanDefinitionBuilder.getBeanDefinition());
    }
}
```

* JedisClientBeanPostProcessor实现了BeanPostProcessor和EnvironmentAware接口，实现EnvironmentAware接口是为了获取到配置文件中相关配置。重写了postProcessBeforeInitialization()，在该方法中，先读取了配置文件中的redis配置，然后为Jedis对象赋值。源码如下:

```java
public class JedisClientBeanPostProcessor implements BeanPostProcessor,EnvironmentAware {

    private static String JEDIS_ADDRESS_PREFIX = "jedis.url";
    private static String JEDIS_PORT_PREFIX = "jedis.port";

    private Environment environment;

    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if(bean instanceof Jedis){
            // 通过beanName获取到namespace
            String prefix = beanName.substring(0, beanName.indexOf(Jedis.class.getSimpleName()));
            // 获取配置文件的配置，配置的规则为： namespace + "." + jedis.url + address|port
            // 示例：demo.jedis.url.address = 127.0.0.1
            String addressKey = prefix + "." + JEDIS_ADDRESS_PREFIX;
            String address = environment.getProperty(addressKey);
            Assert.isTrue(!StringUtils.isEmpty(address),String.format("%s can not be null!!! value = %s",addressKey,address));

            String portKey = prefix + "." + JEDIS_PORT_PREFIX;
            String port = environment.getProperty(portKey);
            Assert.isTrue(!StringUtils.isEmpty(port),String.format("%s can not be null!!! value = = %s",portKey,port));

            // 如果有需要，可以在从配置中添加redis的配置，然后在此处获取即可。
            JedisPool jedisPool = new JedisPool(address,Integer.parseInt(port));
            ((Jedis)bean).setDataSource(jedisPool);
        }
        return bean;
    }
}
```

* 测试
```java
@Configuration
// 开启单机版redis的功能
@EnableJedisClient(namespace = "demo")
// 导入配置文件，如果项目中配置是在Apollo或者SpringCloudConfig等配置中心，则不用导入
@PropertySource("config.properties")
public class AppConfig {
}
```
* 配置文件config.properties
```properties
### 单机版redis配置,配置的前缀注意要和namespace中的值一样
### redis地址和端口号改为自己的即可
demo.jedis.url = 127.0.0.1
demo.jedis.port = 6379
```
* 启动类
```java
public class MainApplication {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);
        System.out.println("============ 测试单机版redis =================");
        Jedis jedis = applicationContext.getBean(Jedis.class);
        String key = "test:key1";
        String value = "value1";
        String writeResult = jedis.setex(key, 3600, value);
        System.out.println("向redis中写入数据，result = " + writeResult);
        String readResult = jedis.get(key);
        System.out.println("从redis中读取数据，result = " + readResult);
    }
}
```

* 控制台打印的结果

### 3.2 @EnableJedisClusterClient
> @EnableJedisClusterClient注解是用来开启redis集群功能的注解。逻辑与@EnableJedisClient一样，只不过针对集群，需要额外做一些处理，需要提供根据key计算槽位，然后根据槽位获取Jedis实例的方法。

* 首先定义一个EnableJedisClusterClient注解，在注解中通过Import注解导入了JedisClusterClientImportRegistrar类，为EnableJedisClusterClient添加了一个属性：namespace，添加该属性的目的是为了让项目中能同时引入多个redis集群。例如：在项目中需要同时连接两个不同的redis集群机器，那么这个时候就可以通过namespace来区分，不通的redis集群指定不同的namespace。
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Import(JedisClusterClientImportRegistrar.class)
public @interface EnableJedisClusterClient {

    String namespace() default "default";
}
```

* JedisClusterClientImportRegistrar 类实现了ImportBeanDefinitionRegistrar接口，重写了registerBeanDefinitions()方法，在方法中向Spring容器中注册了两个Bean，一个是JedisClusterClient，一个是JedisClusterClientBeanPostProcessor后置处理器，该后置处理器是为了在后面JedisClusterClient初始化的过程中，为JedisClusterClient设置连接地址，端口号等属性。
* JedisClusterClient是自定义的一个类，该类持有了对JedisCluster的引用。为什么要自定义这个类呢？因为对于redis集群，我们需要根据槽位来获取jedis对象，不清楚redis集群的朋友，可以先百度查阅一下关于redis集群的知识，后面会写一些关于redis文章的。
* JedisClusterClient在容器中的beanName是 namespace + "JedisClusterClient"，namespace的值是从EnableJedisClusterClient注解中获取到的。例如如下示例使用:那么此时的namespce的值为demo-cluster，如果不指定，则为default。
```java
@EnableJedisClusterClient(namespace = "demo-cluster")
public class AppConfig {
}
```
* JedisClusterClientImportRegistrar源码如下
```java
public class JedisClusterClientImportRegistrar implements ImportBeanDefinitionRegistrar {

    public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry beanDefinitionRegistry) {
        // 注册JedisCluster的后置处理器，用来填充属性
        BeanDefinitionBuilder postProcessorBuilder = BeanDefinitionBuilder.genericBeanDefinition(JedisClusterClientBeanPostProcessor.class);
        beanDefinitionRegistry.registerBeanDefinition(JedisClusterClientBeanPostProcessor.class.getSimpleName(),postProcessorBuilder.getBeanDefinition());

        // 获取namespace，用来指定JedisCluster的beanName
        AnnotationAttributes attributes = AnnotationAttributes.fromMap(annotationMetadata.getAnnotationAttributes(EnableJedisClusterClient.class.getName()));
        String namespace = attributes.getString("namespace");

        // 注册jedisCluster
        BeanDefinitionBuilder clusterBuilder = BeanDefinitionBuilder.genericBeanDefinition(JedisClusterClient.class);
        beanDefinitionRegistry.registerBeanDefinition(namespace+JedisClusterClient.class.getSimpleName(),clusterBuilder.getBeanDefinition());

    }
}
```
* JedisClusterClientBeanPostProcessor实现了BeanPostProcessor和EnvironmentAware接口，在重写的方法中，读取了配置文件中和redis集群相关的配置，然后调用了JedisClusterClient的有参构造方法，new了一个JedisClusterClient。在JedisClusterClient的有参构造方法中，完成了一些对redis集群客户端的初始化操作。
```java
public class JedisClusterClientBeanPostProcessor implements BeanPostProcessor,EnvironmentAware {

    private static String JEDIS_ADDRESS_PREFIX = "jedis.cluster.address";
    private static String JEDIS_MIN_IDEL_PREFIX = "jedis.cluster.minIdel";
    private static String JEDIS_MAX_IDEL_PREFIX = "jedis.cluster.maxIdel";
    private static String JEDIS_MAX_TOTAL_PREFIX = "jedis.cluster.maxTotal";

    private Environment environment;

    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if(bean instanceof JedisClusterClient){
            String namespace = beanName.substring(0,beanName.indexOf(JedisClusterClient.class.getSimpleName()));
            String addressKey = namespace + "." + JEDIS_ADDRESS_PREFIX;
            String address = environment.getProperty(addressKey);
            Assert.isTrue(!StringUtils.isEmpty(address),String.format("%s can not be mull!!!! value = %s",addressKey,address));


            // 可以从配置文件中获取到redis的maxIdle、maxTotal、minIdle等配置，然后封装到poolConfig中
            JedisPoolConfig poolConfig = new JedisPoolConfig();
            Integer minIdel = environment.getRequiredProperty(namespace + "." + JEDIS_MIN_IDEL_PREFIX, Integer.class);
            Integer maxIdel = environment.getRequiredProperty(namespace + "." + JEDIS_MAX_IDEL_PREFIX, Integer.class);
            Integer maxTotal = environment.getRequiredProperty(namespace + "." + JEDIS_MAX_TOTAL_PREFIX, Integer.class);
            poolConfig.setMinIdle(minIdel);
            poolConfig.setMaxIdle(maxIdel);
            poolConfig.setMaxTotal(maxTotal);
            // TODO 还有其他的一些属性，也可以在这儿设置
            JedisClusterClient jedisClusterClient = new JedisClusterClient(address,poolConfig);
            return jedisClusterClient;
        }
        return bean;
    }

    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }
}
```
* JedisClusterClient类实现了JedisClusterCommands,MultiKeyJedisClusterCommands,JedisClusterScriptingCommands这三个接口，这三个接口是redis的jar包提供的类，里面提供了redis的所有和数据相关操作的方法。在JedisClusterClient中我们需要重写这些接口中的方法，由于方法太多，这里只展示一部分代码。
```java
public class JedisClusterClient implements JedisClusterCommands,
        MultiKeyJedisClusterCommands, JedisClusterScriptingCommands {

    private JedisCluster jedisCluster;
    private JedisPoolConfig jedisPoolConfig;
    private JedisSlotBasedConnectionHandler handler;
    private final int defaultConnectTimeout = 2000;
    private final int defaultConnectMaxAttempts = 20;

    /**
     * 为什么在这里提供一个无参构造器呢？
     * 因为在Spring在实例化bean时，是推断出类的构造器，然后根据类的构造器来反射创建bean，
     * 如果不提供默认的无参构造器，那么Spring就会使用JedisClusterClient的有参构造器。
     * 然而，有参构造器中需要namespace,address,poolConfig等参数。
     * 此时，Spring就会从Spring容器中根据参数的类型去获取Bean，获取不到就会报错。
     * 所以这里特意提供了一个无参构造器
     */
    public JedisClusterClient(){}

    public JedisClusterClient(String address, JedisPoolConfig poolConfig) {
        this.jedisPoolConfig = poolConfig;
        // 解析redis配置的地址
        String[] addressArr = address.split(",");
        Set<HostAndPort> hostAndPortSet = new HashSet<HostAndPort>(addressArr.length);
        for (String url : addressArr) {
            String[] split = url.split(":");
            String host = split[0];
            int port = Integer.parseInt(split[1]);
            hostAndPortSet.add(new HostAndPort(host,port));
        }
        // 实例化jedisCluster
        this.jedisCluster = new JedisCluster(hostAndPortSet,defaultConnectTimeout,defaultConnectMaxAttempts,jedisPoolConfig);

        try {
        	// 根据反射获取到connectionHandler的值
        	// 目的是为了在后面通过它根据槽位获取redis实例
            Field connectionHandlerField = BinaryJedisCluster.class.getDeclaredField("connectionHandler");
            connectionHandlerField.setAccessible(true);
            this.handler  = (JedisSlotBasedConnectionHandler) connectionHandlerField.get(jedisCluster);
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }

    /**
     * 在redis集群中，根据key计算出key在哪一个slot，然后获取该slot所属于哪一个台redis机器
     * @param key
     * @return
     */
    public Jedis getResource(String key){
        int slot = JedisClusterCRC16.getSlot(key);
        return handler.getConnectionFromSlot(slot);
    }


    public String set(String key, String value) {
        return jedisCluster.set(key,value);
    }

    public String set(String key, String value, SetParams params) {
        return jedisCluster.set(key, value, params);
    }

    public String get(String key) {
        return jedisCluster.get(key);
    }

    // 其他的方法都是直接调用jedisCluster对应的方法即可
}
```

* 测试redis集群
```java
@Configuration
// 开启集群版redis的功能，namespace指定为demo-cluster，所以配置连接地址等属性时，需要以demo-cluster为前缀
@EnableJedisClusterClient(namespace = "demo-cluster")
@PropertySource("config.properties")
public class AppConfig {
}
```
* 配置文件config.properties
```properties
demo-cluster.jedis.cluster.address = redis001:6379,redis003:6379,redis003:6379
demo-cluster.jedis.cluster.minIdel = 1
demo-cluster.jedis.cluster.maxIdel = 10
demo-cluster.jedis.cluster.maxTotal = 30
```
* 启动类
```java
public class MainApplication {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(AppConfig.class);

        // 测试jedis-cluster
        System.out.println("============= 测试集群版redis ================");
        JedisClusterClient jedisCluster = applicationContext.getBean(JedisClusterClient.class);
        String clusterKey = "test:{1000005}";
        String clusterValue = "" + System.currentTimeMillis();
        String res1 = jedisCluster.setex(clusterKey, 3600, clusterValue);
        System.out.println("向redis集群中写数据，result = " + res1);
        String res2 = jedisCluster.get(clusterKey);
        System.out.println("从redis集群中获取数据，result = " + res2);

        // 测试redis事务
        System.out.println("============= 测试redis集群事务 ==============");
        Jedis resource = jedisCluster.getResource(clusterKey);
        try {
            if(resource.watch(clusterKey).equalsIgnoreCase("OK")){
                Transaction transaction = resource.multi();
                String tmp = System.currentTimeMillis() + "";
                System.out.println("tmp = " + tmp);
                transaction.setex(clusterKey, 3600, tmp);
                List<Object> exec = transaction.exec();
                System.out.println(exec);
            }
        }finally {
           if(resource != null){
               resource.unwatch();
               resource.close();
           }
        }
        // 在此获取clusterKey的值，验证通过事务是否更新了缓存中的值
        System.out.println("after watch, result = " + jedisCluster.get(clusterKey));

    }
}
```
* 控制台打印结果


## 4. 总结
* 本文通过自定义两个注解@EnableJedisClient和@EnableJedisClusterClient，实现了Redis和Spring的整合，并对其进行了测试。
* 代码中存在的不足之处，因为只是简易版的插件，所以在两个后置处理中，对配置属性的读取和赋值，不够灵活，例如在本文中，想要对redis客户端的test-on-return属性配置指定值，则在本文是没有实现的。解决办法可以是，利用SpringBoot中的Binder来实现属性的绑定，这样只要在配置文件中配置了，就能够对JedisPool中对应的属性赋值了。
* 另外在本文中是利用了两个BeanPostProcessor对Jedis和JedisCluster进行了初始化操作，实际上还可以有其他的解决办法。例如：在通过ImportBeanDefinitionRegistrar向容器中注册Bean时，我们可以注册一个FactoryBean，如自定义一个JedisFactoryBean。然后在JedisFactoryBean的getObject()方法中去完成对Jedis的初始化操作。这样在第一次从容器中获取Jedis对象的时候，就会调用到JedisFactoryBean的getObject()方法，这样Jedis就完成了初始化操作。（注意：FactoryBean的getObject()方法返回的Bean，是在第一次获取Bean的时候才进行的实例化操作，实例化完成后，会放入到Spring的一个缓存中），关于FactoryBean的知识，感兴趣的朋友可以先自行百度一下，后面也会单独写文章分析。

## 5. 源码地址
* 本文中所有源码都已上传至GitHub。地址为：https://github.com/TianTang2013/demo-source.git
* 或者点击此处跳转：[跳转至GitHub](https://github.com/TianTang2013/demo-source.git)


## 6. 推荐

> 最后推荐一款本人所在公司开源的性能监控工具——`Pepper-Metrics`

* 地址： https://github.com/zrbcool/pepper-metrics
* 或者 [点击此处跳转](https://github.com/zrbcool/pepper-metrics)
* `Pepper-Metrics`是坐我对面的两位同事一起开发的开源组件，主要功能是通过比较轻量的方式与常用开源组件（`jedis/mybatis/motan/dubbo/servlet`）集成，收集并计算`metrics`，并支持输出到日志及转换成多种时序数据库兼容数据格式，配套的`grafana dashboard`友好的进行展示。项目当中原理文档齐全，且全部基于`SPI`设计的可扩展式架构，方便的开发新插件。另有一个基于`docker-compose`的独立`demo`项目可以快速启动一套`demo`示例查看效果`https://github.com/zrbcool/pepper-metrics-demo`。如果大家觉得有用的话，麻烦给个`star`，也欢迎大家参与开发，谢谢：）

> 扫描下方二维码即可关注微信公众号`菜鸟飞呀飞`，一起阅读更多Spring源码。
