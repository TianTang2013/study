> 点击上方`菜鸟飞呀飞`或者扫描下方二维码，即可关注微信公众号。

【图】

> IOC模块可以算得上是Spring中最核心的功能了，它完成了Bean的自动装配。在我们的业务代码中，经常会出现A类的实例中需要注入B类的实例，B类的实例需要注入A类的实例，这种情况我们称之为循环依赖。按照我们通常的想法，这个时候就应该出现死循环了，无法完成依赖注入过程。而在实际项目中，在项目启动时系统并没有报错，这是为什么呢？答案就是Spring在自动装配的过程中为我们处理了循环依赖的这种情况。那么Spring是如何处理循环依赖的问题的呢？

## 循环依赖

* 首先我们定义如下两个类：UserService和OrderService，并让它们相互依赖。
```java
@Service
public class UserService {

    @Autowired
    private OrderService orderService;

    public void query(){
        System.out.println(orderService);
    }
}
```
```java
@Service
public class OrderService {

    @Autowired
    private UserService userService;

    public void query(){
        System.out.println(userService);
    }
}
```
> 在分析Spring如何解决循环依赖之前，建议读者先看下Spring的refresh()方法的流程和Bean的创建过程，可以参考笔者的另外两篇文章。[通过源码看Bean的创建过程](https://mp.weixin.qq.com/s/WwjicbYtcjRNDgj2bRuOoQ)和[Spring源码系列之容器启动流程](https://mp.weixin.qq.com/s/q6zs7xRjpcB4YxLw6w477w)。

* 1. Spring在启动时，会在refresh()方法中创建所有的单例Bean，并完成自动装配。对于上面我们定义的两个类，假如先创建OrderService，那么就会先调用`getBean(orderService)`，getBean()方法会调用到doGetBean()。
* 2. 在doGetBean()方法中会先调用`getSingleton(orderService)`方法，判断它的返回值是否为空（此时第一次调用，返回值一定为空），如果不为空，则返回bean，如果为空，则继续执行下面的逻辑。在后面又会调用到getSingleton()的一个重载方法：`getSingleton(orderService,lambda表达式)`，在这个方法中会调用createBean()方法。doGetBean()简化后的代码如下：
```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
        @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

    final String beanName = transformedBeanName(name);
    Object bean;
    // 调用getSingleton()，第一次调用的时候，返回值肯定为空，因为此时orderService还没有被创建
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }else {
        try {
            if (mbd.isSingleton()) {
                // 调用getSingleton()的重载方法，第二个参数是一个lambda表达式
                sharedInstance = getSingleton(beanName, () -> {
                    try {
                        return createBean(beanName, mbd, args);
                    }
                    catch (BeansException ex) {
                        throw ex;
                    }
                });
            }
        }
        catch (BeansException ex) {
            throw ex;
        }
    }
    return (T) bean;
}
```
* 3. createBean()方法会调用到doCreateBean()方法，在doCreateBean()方法中，会先实例化Bean（此时的Bean是一个半成品），然后将Bean通过`addSingletonFactory()`方法放入到singletonFactories属性中，接着调用populateBean()方法为Bean填充属性，完成自动装配。例如：先创建orderService，然后将其放到`singletonFactories`中，接着通过populateBean()方法为orderService装配userService属性。
* 4. 在为orderService装配userService属性时，由于userService此时还没有被创建，所以又会调用到`getBean(userService)`。既然调用到getBean()方法，那么就会`同上面的逻辑一样`，先调用`getSingleton(userService)判断是否为空`，此时`仍然为空`，因为是第一次获取userService，所以接着会调用到createBean()、doCreateBean()。
* 5. 当进入到doCreateBean()当中时，同样也是先实例化userService，然后`将半成品的userService放入到singletonFactories中`，接着通过populateBean()方法来为userService装配orderService属性。
* 6. 在为userService装配orderService属性时，又会调用到`getBean(orderService)`方法，然后调用doGetBean(orderService)。`接下来一步就与之前不一样了`，在doGetBean(orderService)中仍然是先调用getSingleton(orderService)方法，`此时该方法不会返回null了`，因为在第3步中，`将orderService放入到了singletonFactories中`，所以此时返回值不为空，那么doGetBean()方法就会将bean返回，`不会再调用到createBean()方法去创建orderService了`。getSingleton()方法的源码如下:
```java
public Object getSingleton(String beanName) {
    return getSingleton(beanName, true);
}
```
```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 先从singletonObjects获取
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized (this.singletonObjects) {
            // 然后从earlySingletonObjects中获取
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                // 最后从singletonFactories中获取
                ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```
* 7. 可以看到在getSingleton(beanName,allowEarlyReference)中，会先从`singletonObjects`中获取bean，如果获取不到，就从`earlySingletonObjects`中获取，如果还获取不到，就从`singletonFactories`中获取，所以当第一次调用getSingleton(beanName,allowEarlyReference)时，肯定返回null，只有当将bean放入到singletonFactories中了，才会不为空。由于在创建orderService时，就已经将orderService放入到singletonFactories中了，所以此时会将orderService返回。
* 8. 当将orderService对象返回后，代码就会回到userService对象的populateBean()方法中，此时就能将其赋值给userService对象中的orderService属性了。（注意此时的orderService对象仍然是一个半成品，这个对象中的属性还没有被填充，但是没有关系，因为userService对象持有的是orderService的引用，所以当后面orderService的属性变化了，对userService是没有影响的。）
* 9. 接着进行userService对象的其他属性填充，当userService完全初始化完成后，先将userService对象放入到singletonObjects中，再将userService对象返回。然后代码就会回到orderService对象的populateBean()方法处了，此时就可以将userService对象赋值给orderService对象的userService属性了，接着继续完成orderService的初始化操作。最后将orderService返回，然后将其放入到singletonObjects中。这样就完成了对象之前循环依赖。
* 整体流程可以参考如下流程图：
[图]
* 从上面的分析中可以发现，解决循环依赖的主要功臣是getSingleton(beanName)、addSingletonFactory()这两个方法和singletonFactories、earlySingletonObjects、singletonObjects这三个属性。在Bean实例化之后属性填充之前，先将Bean通过addSingletonFactory()方法放入到singletonFactories中，当出现循环依赖时，先通过getSingleton(beanName)依次从singletonObjects、earlySingletonObjects、singletonFactories这三个属性中获取Bean，如果bean存在，则返回。由于是对象引用，所以即使早期装配的是半成品的Bean，也没有影响。

## 总结
* 本文通过举例UserService和OrderService，分析了Spring解决循环依赖的过程，最后通过一张流程图进行了总结。
* 关于singletonFactories、earlySingletonObjects、singletonObjects这三个属性，最后补充说明一下，singletonFactories中存放的bean是通过构造器反射创建出来的bean，没有进行任何属性填充，是个半成品bean；earlySingletonObjects中存放的也是半成品，只不过在singletonFactories的基础上多了一次Bean后置处理器的处理；singletonObjects中存放的是最后完整的Bean，我们通常所说的Spring的Bean容器，可以简单理解为就是singletonObjects。
* 关于本文中提到的Spring容器的refresh()方法以及getBean()、createBean()方法，详细的源码分析可以参考笔者的另外两篇文章。
* [通过源码看Bean的创建过程](https://mp.weixin.qq.com/s/WwjicbYtcjRNDgj2bRuOoQ)
* [Spring源码系列之容器启动流程](https://mp.weixin.qq.com/s/q6zs7xRjpcB4YxLw6w477w)

## 相关推荐
* [Spring中最！最！最！重要的后置处理器！没有之一！！！](https://mp.weixin.qq.com/s/f2vSH9YNmnNqdps05LEEHw)
* [@Import和@EnableXXX](https://mp.weixin.qq.com/s/y_2Z9m0gevp-cMkEIflrwA)
* [手写一个Redis和Spring整合的插件](https://mp.weixin.qq.com/s/AU0QpzD0xNslgeWEJ6ujQg)
* [为什么JDK的动态代理要基于接口实现而不能基于继承实现？](https://mp.weixin.qq.com/s/vLnjd80q9q1SNZy6yvzqYw)
* [FactoryBean——Spring的扩展点之一](https://mp.weixin.qq.com/s/NewVzdhA_BNq-LtOahxSAQ)
* [@Autowired注解的实现原理](https://mp.weixin.qq.com/s/qNuGgzPiOha0e1tCW46e8Q)
* [一次策略设计模式的实际应用](https://mp.weixin.qq.com/s/DOBnL1q6UMpWrcrKVvOkdw)
* [管程:并发编程的基石](https://mp.weixin.qq.com/s/6jvA5jnnMkr5l-IliDL8yw)




