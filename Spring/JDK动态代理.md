> 点击上方`菜鸟飞呀飞`，即可关注微信公众号。

[TOC]

## 1. 问题
> 在阅读本文之前，可以先思考一下下面几个问题。

* 为什么说JDK的动态代理要基于接口实现，而不能基于继承来实现？
* 在JDK的动态代理中，在目标对象方法内部调用自己的另一个方法时，另一个方法在执行时，为什么没有经过代理对象？


## 2. JDK的动态代理的固定写法
* JDK的动态代理的写法比较固定，需要先定义一个接口和接口的实现类，然后再定义一个实现了InvocationHandler接口的实现类。然后调用Proxy类的newInstance()方法即可。示例代码如下：
* 先定义一个接口：UserService，接口中有两个方法
```java
public interface UserService {

    int insert();

    String query();
}
```
* 再定义一个UserService接口的实现类：UserServiceImpl
```java
public class UserServiceImpl implements UserService{

    @Override
    public int insert() {
        System.out.println("insert");
        return 0;
    }

    @Override
    public String query() {
        System.out.println("query");
        return null;
    }
}
```
* 再定义一个InvocationHandler接口的实现类：UserServiceInvocationHandler。在自定义的InvocationHandler中，定义了一个属性：target，定义这个属性的目的是为了在InvocationHandler中持有对目标对象的引用，target属性的初始化是在构造器中进行初始化的。
```java
public class UserServiceInvocationHandler implements InvocationHandler {

    // 持有目标对象
    private Object target;

    public UserServiceInvocationHandler(Object target){
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("invocation handler");
        // 通过反射调用目标对象的方法
        return method.invoke(target,args);
    }
}
```
* 通过Proxy.newProxyInstance()方法创建代理对象
```java
public class MainApplication {

    public static void main(String[] args) {
        // 指明一个类加载器，要操作class文件，怎么少得了类加载器呢
        ClassLoader classLoader = MainApplication.class.getClassLoader();
        // 为代理对象指定要是实现哪些接口，这里我们要为UserServiceImpl这个目标对象创建动态代理，所以需要为代理对象指定实现UserService接口
        Class[] classes = new Class[]{UserService.class};
        // 初始化一个InvocationHandler，并初始化InvocationHandler中的目标对象
        InvocationHandler invocationHandler = new UserServiceInvocationHandler(new UserServiceImpl());
        // 创建动态代理
        UserService userService = (UserService) Proxy.newProxyInstance(classLoader, classes, invocationHandler);
        // 执行代理对象的方法，通过观察控制台的结果，判断我们是否对目标对象(UserServiceImpl)的方法进行了增强
        userService.insert();
    }
}
```
* 控制台打印结果如下，从打印结果来看，已经完成了对目标对象UserServiceImpl的代理（增强）。
[图]


## 3. 如何查看代理类的class文件
> 阅读到这里，我们基本知道了JDK动态代理的写法，功能虽然实现了，但是对于喜欢研究源码的同学，可能会思考：为什么调用Proxy.newProxyInstance()方法后，就产生了代理对象，对目标对象的方法进行了增强？原理是什么呢？

* 通常学习某项技术的原理，都是看源文件.java或者借助反编译工具反编译.class文件后去分析。可是对于JDK的动态代理，它产生的代理对象是在运行时创建的，通过常规操作，我们没办法得到这个代理对象对应的.class文件。如果能得到代理对象多对应的class文件，那动态代理的原理，我们就好分析了。
* 所以接下来介绍两种获取到代理对象的.class文件的方法

### 3.1 手动写到磁盘

* 在调用Proxy.newProxyInstance()方法时，最终会调用到ProxyGenerator.generateProxyClass()方法，该方法的作用就是生成代理对象的class文件，返回值是一个byte[]数组。所以我们可以将生成的byte[]数组通过输出流，将内容写出到磁盘。
```java
public static void main(String[] args) throws IOException {
    String proxyName = "com.tiantang.study.$Proxy0";
    Class[] interfaces = new Class[]{UserService.class};
    int accessFlags = Modifier.PUBLIC;
    byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
            proxyName, interfaces, accessFlags);
    // 将字节数组写出到磁盘
    File file = new File("/Users/liujinkun/Downloads/dynamic/$Proxy0.class");
    OutputStream outputStream = new FileOutputStream(file);
    outputStream.write(proxyClassFile);
}
```
* 运行完main()方法后，从目录中`/Users/liujinkun/Downloads/dynamic/$Proxy0.class`找到生成的文件，由于是一个class文件，所以我们需要把它有反编译器编译一下，例如：在idea中，将文件放到target目录下，打开文件就能看到反编译后的代码了。

### 3.2 自动写到磁盘
* 第二种方法是通过百度查来的，在代码中添加一行代码：`System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true")`，就能实现将程序运行过程中产生的动态代理对象的class文件写入到磁盘。如下示例：
```java
public class MainApplication {

    public static void main(String[] args) {
        // 让代理对象的class文件写入到磁盘
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
        // 指明一个类加载器，要操作class文件，怎么少得了类加载器呢
        ClassLoader classLoader = MainApplication.class.getClassLoader();
        // 为代理对象指定要是实现哪些接口，这里我们要为UserServiceImpl这个目标对象创建动态代理，所以需要为代理对象指定实现UserService接口
        Class[] classes = new Class[]{UserService.class};
        // 初始化一个InvocationHandler，并初始化InvocationHandler中的目标对象
        InvocationHandler invocationHandler = new UserServiceInvocationHandler(new UserServiceImpl());
        // 创建动态代理
        UserService userService = (UserService) Proxy.newProxyInstance(classLoader, classes, invocationHandler);
        // 执行代理对象的方法，通过观察控制台的结果，判断我们是否对目标对象(UserServiceImpl)的方法进行了增强
        userService.insert();
    }
}
```
* 运行程序，最终发现在项目的根目录下出现了一个包：com.sun.proxy。包下有一个文件`$Proxy0.class`。在idea打开，发现就是所产生代理类的源代码。

## 4. 问题解答
> 通过上面两种方法获取到代理对象的源代码如下：
```java
package com.sun.proxy;

import com.tiantang.study.UserService;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;

public final class $Proxy0 extends Proxy implements UserService {
    private static Method m1;
    private static Method m3;
    private static Method m4;
    private static Method m2;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final int insert() throws  {
        try {
            return (Integer)super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String query() throws  {
        try {
            return (String)super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m3 = Class.forName("com.tiantang.study.UserService").getMethod("insert");
            m4 = Class.forName("com.tiantang.study.UserService").getMethod("query");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```
* 通过源码我们发现，`$Proxy0`类继承了Proxy类，同时实现了UserService接口。到这里，我们的问题一就能解释了，为什么JDK的动态代理只能基于接口实现，不能基于继承来实现？因为Java中不支持多继承，而JDK的动态代理在创建代理对象时，默认让代理对象继承了Proxy类，所以JDK只能通过接口去实现动态代理。

* `$Proxy0`实现了UserService接口，所以重写了接口中的两个方法（`$Proxy0`同时还重写了Object类中的几个方法）。所以当我们调用query()方法时，先是调用到`$Proxy0.query()`方法，在这个方法中，直接调用了super.h.invoke()方法，父类是Proxy，父类中的`h`就是我们定义的InvocationHandler，所以这儿会调用到UserServiceInvocationHandler.invoke()方法。因此当我们通过代理对象去执行目标对象的方法时，会先经过InvocationHandler的invoke()方法，然后在通过反射method.invoke()去调用目标对象的方法，因此每次都会先打印`invocation handler`这句话。
```java
public class UserServiceInvocationHandler implements InvocationHandler {

    // 持有目标对象
    private Object target;

    public UserServiceInvocationHandler(Object target){
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("invocation handler");
        // 通过反射调用目标对象的方法
        return method.invoke(target,args);
    }
}
```
* 对于文章开头的第二个问题，我们先来看下如下示例代码
```java
public class UserServiceImpl implements UserService{

    @Override
    public int insert() {
        System.out.println("insert");
        query();
        return 0;
    }

    @Override
    public String query() {
        System.out.println("query");
        return null;
    }
}
```
```java
public class MainApplication {

    public static void main(String[] args) {
        // 让代理对象的class文件写入到磁盘
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
        // 指明一个类加载器，要操作class文件，怎么少得了类加载器呢
        ClassLoader classLoader = MainApplication.class.getClassLoader();
        // 为代理对象指定要是实现哪些接口，这里我们要为UserServiceImpl这个目标对象创建动态代理，所以需要为代理对象指定实现UserService接口
        Class[] classes = new Class[]{UserService.class};
        // 初始化一个InvocationHandler，并初始化InvocationHandler中的目标对象
        InvocationHandler invocationHandler = new UserServiceInvocationHandler(new UserServiceImpl());
        // 创建动态代理
        UserService userService = (UserService) Proxy.newProxyInstance(classLoader, classes, invocationHandler);
        // 执行代理对象的方法，通过观察控制台的结果，判断我们是否对目标对象(UserServiceImpl)的方法进行了增强
        userService.insert();
    }
}
```
* 在这段代码中，只修改了UserServiceImpl的insert()方法，在打印之后，我们调用了一下query()方法。在看打印结果之前，我们先来分析下打印结果：按照我们的理解`$Proxy0`代理类对UserServiceImpl中的方法都进行了增强，每次调用UserServiceImpl中类的方法时，应该都会经过InvocationHandler中的invoke()方法，每次都会打印`invocation handler`这句话。所以当我们调用`userService.insert()`时，打印结果应该是：
```java
invocation handler
insert
invocation handler
query
```
* 然后，结果真的如我们想象的那样吗？下面截图为程序运行的结果：
[图]

* 从结果中，我们发现，只打印了一次`invocation handler`。在调用query()方法时，并没有去执行InvocationHandler中的invoke()方法。为什么呢？原因就在这个地方：
```java
public int insert() {
    System.out.println("insert");
    query();
    return 0;
}
```
* 在insert()方法中调用query()方法时，实际上调用的是this.query()，而此时this对象是UserServiceImpl，并不是`$Proxy0`这个代理对象，只有在调用代理对象的query()方法时，才会经过InvocationHandler.invoke()方法，所以此时只会打印一次`invocation handler`。所以第二个问题的答案就是：在调用过程中，this的指向发生了变化。

## 5. 总结与思考
* 本文介绍了JDK动态代理的常规用法，然后通过特殊手段，获取到了代理对象的class文件，然后根据class文件，分析了JDK动态代理中两个比较常见的问题。
* 看到这里相信你已经明白了JDK动态代理的原理，以及JDK动态代理所带来的问题——在目标对象的方法A内部调用目标对象的其他方法B，此时动态代理并不会对方法B进行增强。
* 相信很多人曾经碰到过在Spring项目中，通过@Transactional来声明事务时，出现了事务不生效的情况，结合本文，现在相信你应该能明白这其中的问题了吧？（关于Spring事务失效的例子，可以百度搜索: JDK 动态代理 事务失效）
* 同样的案例还有@Async等案例，在同一个类的两个方法中开启异步，然后在方法内部进行互相调用，最终导致异步失效的问题。
* 既然JDK动态代理存在这些问题，那现在Spring中是如何避免这些问题的呢？当然是大名鼎鼎的CGLIB了，关于CGLIB，下一篇文章分析。

## 6. 猜你喜欢
* [通过源码看Bean的创建过程](https://mp.weixin.qq.com/s/WwjicbYtcjRNDgj2bRuOoQ)
* [Spring源码系列之容器启动流程](https://mp.weixin.qq.com/s/q6zs7xRjpcB4YxLw6w477w)
* [Spring中最！最！最！重要的后置处理器！没有之一！！！](https://mp.weixin.qq.com/s/f2vSH9YNmnNqdps05LEEHw)
* [@Import和@EnableXXX](https://mp.weixin.qq.com/s/y_2Z9m0gevp-cMkEIflrwA)
* [手写一个Redis和Spring整合的插件](https://mp.weixin.qq.com/s/AU0QpzD0xNslgeWEJ6ujQg)