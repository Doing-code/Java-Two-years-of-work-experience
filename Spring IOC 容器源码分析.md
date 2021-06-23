---
typora-copy-images-to: Spring IOC 图集
---

> 原文出处 ：https://javadoop.com/post/spring-ioc#toc_13

本文采用的源码版本是 4.3.11.RELEASE，但部分代码是直接拷贝的 5.x版本的源码

## 引言

先看下最基本的启动 Spring 容器的例子 ：

```java
public static void main(String[] args) {
    ApplicationContext context = new ClassPathXmlApplicationContext("classpath:applicationfile.xml");
}
```

以上代码就可以利用配置文件来启动一个 Spring 容器了。

如果是 maven 工程，加上以下依赖即可 ：spring 版本自定义

```pom
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-context</artifactId>
  <version>4.3.11.RELEASE</version>
</dependency>
```

> spring-context 会自动将 spring-core、spring-beans、spring-aop、spring-expression 这几个基础 jar 包带进来。

<br/>

`ApplicationContext context = new ClassPathXmlApplicationContext(...)` 其实很好理解，从名字上就可以猜出一二，就是在 ClassPath

 中寻找 xml 配置问及那，根据 xml 文件内容来构建 ApplicationContext 。当然了，除了 ClassPathXmlApplicationContext 以外，我们也还有其他构建 ApplicationContext 的方案可供选择，我们先来看看大体的继承结构是怎么样的：

![1](SpringIOC图集/1.png)

我们可以看到 ，ClassPathXmlApplicationContext  兜兜转转好久才到 ApplicationContext 接口，同样的，我们也可以使用绿色的  `FileSystemXmlApplicationContext` 和 `AnnotationConfigApplicationContext` 这两个类 。

1、FileSystemXmlApplicationContext 的构造函数需要一个 xml 配置文件在系统中的路径 ，比如 "C://xxx/xxx/application.xml" 。和 ClassPathXmlApplicationContext 基本上一样 。

2、AnnotationConfigApplicationContext 是基于注解来使用的，它不需要配置文件，采用 Java 配置类和各种注解来配置，是比较简单的方式，也是大势所趋吧 。

我们先来看一个简单的例子是怎么实例化 ApplicationContext 。

定义一个接口 ：

```java
public interface MessageService {
    String getMessage();
}
```

定义接口实现类 ：

```java
public class MessageServiceImpl implements MessageService {

    public String getMessage() {
        return "hello world";
    }
}
```

然后在 工程下 resources 目录新建一个配置文件，文件名随意，通常叫 application.xml 或 application-xxx.xml 就可以了：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns="http://www.springframework.org/schema/beans"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd" default-autowire="byName">

    <bean id="messageService" class="com.javadoop.example.MessageServiceImpl"/>
</beans>
```

这样，就可以跑起来了：

```java
public class App {
    public static void main(String[] args) {
        // 用我们的配置文件来启动一个 ApplicationContext
        ApplicationContext context = new ClassPathXmlApplicationContext("classpath:application.xml");

        System.out.println("context 启动成功");

        // 从 context 中取出我们的 Bean，而不是用 new MessageServiceImpl() 这种方式
        MessageService messageService = context.getBean(MessageService.class);
        // 这句将输出: hello world
        System.out.println(messageService.getMessage());
    }
}
```

由此引出本文的主题 ，怎=怎样通过配置文件来启动 Spring 的 ApplicationContext？这也是我们今天要分析的 IOC 的核心。ApplicationContext 启动过程中，会负责创建实例 Bean，往各个 Bean 中注入依赖等 。





## BeanFactory 简介

BeanFactory ，从名字上也很好理解，生产 bean 的工厂。它负责生产和管理各个 bean 实例 。

前面说的 ApplicationContext 其实就是一个 BeanFactory。我们来看一下 BeanFactory 接口相关的主要继承结构 ：

![2](SpringIOC图集/2.png)

类图如下 ：

![image-20210622200953295](SpringIOC图集/image-20210622200953295.png)

重点说明 ：

- ApplicationContext 继承了 ListableBeanFactory，这个 Listable 的意思就是，通过这个接口，我们可以获取多个 bean，大家阅读源码会发现，最顶层的 BeanFactory 接口的方法都是获取单个 Bean 的 。
- ApplicationContext 继承了 HierarchicalBeanFactory，Hierarchical单词本身已经能够说明问题了，也就是说我们可以在应用中起多个 BeanFactory ，然后可以将各个 BeanFactory 设置为 父子关系 。
- AutowireCapableBeanFactory 这个类名中的 Autowire 大家应该都非常的熟悉，它是用来自动装配 Bean 用的，但是仔细看上图，ApplicationContext 并没有继承它，不过不用担心，不使用继承，但不代表不可以使用组合。如果你看到 ApplicationContext 接口定义中的最后一个方法 getAutowireCapableBeanFactory()  就知道了 。
- ConfigurableListableBeanFactory 也是一个特殊的接口，看图，特殊之处在于它继承了第二层所有的三个接口，而 ApplicationContext 没有。这点之后会用到 。
- 请先不用花时间在其他的接口和类上，先理解我说的这几点就可以了 。

然后，请读者打开编辑器，翻一下 BeanFactory、ListableBeanFactory、HierarchicalBeanFactory、AutowireCapableBeanFactory、ApplicationContext 这几个接口的代码，大概看一下各个接口中的方法，大家心里要有底



## 启动过程分析

第一步 ，我们从 `ClassPathXmlApplicationContext` 的构造方法说起 。

```java
public class ClassPathXmlApplicationContext extends AbstractXmlApplicationContext {
    private Resource[] configResources;

    // 如果已经有 ApplicationContext 并需要配置成父子关系，那么调用这个构造方法 。 
    public ClassPathXmlApplicationContext(ApplicationContext parent) {
        super(parent);
    }

    public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
        this(new String[]{configLocation}, true, (ApplicationContext)null);
    }
     public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, @Nullable ApplicationContext parent) throws BeansException {
          super(parent);
          // 根据提供的路径，处理成配置文件数组(以分号、逗号、空格、tab、换行符分割)
          this.setConfigLocations(configLocations);
          if (refresh) {
               // 核心方法 
               this.refresh();
          }

     }
}
```

接下来，就是 `refresh()` ，这里简单说一下为什么是 refresh()，而不是 init() 这种名字的方法 。因为 ApplicationContext 建立起来以后，其实我们是可以通过调用 refresh() 方法重建的，refresh() 会将原来的 ApplicationContext 销毁，然后再重新执行一次初始化操作 。

往下看，refresh() 方法调用了那么多方法，就知道肯定不简单。

```java
// AbstractApplicationContext.java
@Override
public void refresh() throws BeansException, IllegalStateException {
     // 来个锁, 不然 refresh() 方法还没执行完，你又来一个启动或销毁容器的操作，那不就乱套了嘛
     synchronized(this.startupShutdownMonitor) {
          
          // 准备工作，记录一下容器的启动时间，标记"已启动"状态，处理配置文件中的占位符
          this.prepareRefresh();
          
          /**
             这步比较关键，这步完成后，配置文件就会解析成一个个 Bean 定义，注册到 BeanFactory 中，
             当然，这里说的 Bean 还没有初始化，只是配置信息都提取出来了 。
             注册也只是将这些信息保存到了注册中心(说到底核心是一个 beanName-> beanDefinition 的 map)
          */
          ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
          
          /**
             设置 BeanFactory 的来加载器，添加几个 BeanPostProcessor，手动注册的几个特殊的 bean
             这块待会会展开说
          */
          this.prepareBeanFactory(beanFactory);

          try {
               /**
                  【这里需要知道 BeanFactoryPostProcessor 这个知识点，Bean 如果实现了此接口，
                  那么在容器初始化以后，Spring 会负责调用里面的 postProcessBeanFactory() 方法。】
               */
               
               /**
                  这里是提供给子类的扩展点，到这里的时候，所有的 Bean 都加载、注册完成了，但是都还没有初始化
                  具体的子类可以在这步的时候添加一些特殊的 BeanFactoryPostProcessor 的实现类或做点什么事
               */
               this.postProcessBeanFactory(beanFactory);
               // 调用 BeanFactoryPostProcessor 各个实现类的 postProcessBeanFactory(factory) 方法
               this.invokeBeanFactoryPostProcessors(beanFactory);
               
               /**
                  注册 BeanPostProcessor 的实现类，注意看和 BeanFactoryPostProcessor 的区别
                  此接口的两个方法：postProcessBeforeInitialization 和 postProcessAfterInitialization
                  两个方法分别在 Bean 初始化之前和初始化之后得到执行。注意，到这里 Bean 还没初始化 。
               */
               this.registerBeanPostProcessors(beanFactory);
               
               // 初始化当前 ApplicationContext 的 MessageSource，国际化这里不展开说了，不然没完没了了。
               this.initMessageSource();
               
               // 初始化当前 ApplicationContext 的事件广播器，这里也不展开了
               this.initApplicationEventMulticaster();
               
               /**
                  从方法名就可以知道，典型的模板方法(钩子方法)，
                  具体的子类可以在这里初始化一些特殊的 Bean (在初始化 singleton beans 之前)
               */
               this.onRefresh();
               
               // 注册事件监听器，监听器需要实现 ApplicationListener 接口。这也不是我们的重点，过
               this.registerListeners();
               
               /**
                  重点,重点,重点,
                  初始化所有的 singleton benas
                  (lazy-init 的除外) (懒加载)
               */
               this.finishBeanFactoryInitialization(beanFactory);
               
               // 最后，广播事件，ApplicationContext 初始化完成 
               this.finishRefresh();
          } catch (BeansException var9) {
               if (this.logger.isWarnEnabled()) {
                    this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
               }

               // Destroy already created singletons to avoid dangling resources
               // 销毁已经初始化的 singleton 的 Beans，以免有些 bean 会一直占用资源
               this.destroyBeans();
               this.cancelRefresh(var9);
               
               // 异常往外抛
               throw var9;
          } finally {
               this.resetCommonCaches();
          }

     }
}
```

下面，我们开始一步步来深入这个 refresh() 方法 。

### 创建 Bean 容器前的准备工作

这个比较简单，直接看代码中的注释即可。

```java
protected void prepareRefresh() {
     /**
        记录启动事件，将 closed 属性设置为 false，active属性设置为 true，它们都是 AtomicBoolean 类型 
     */
     this.startupDate = System.currentTimeMillis();
     this.closed.set(false);
     this.active.set(true);
     if (this.logger.isDebugEnabled()) {
          if (this.logger.isTraceEnabled()) {
               this.logger.trace("Refreshing " + this);
          } else {
               this.logger.debug("Refreshing " + this.getDisplayName());
          }
     }

     // Initialize any placeholder property sources in the context environment (初始化上下文环境中的任何占位符属性源)
     this.initPropertySources();
     
     // 校验 xml 配置文件
     this.getEnvironment().validateRequiredProperties();
     this.earlyApplicationEvents = new LinkedHashSet();
}
```



### 创建 Bean 容器，加载并注册 Bean

我们回到 refresh() 方法中的下一行 this.obtainFreshBeanFactory();

注意，这个方法事全文最重要的部分之一，这里将会初始化 BeanFactory、加载 Bean、注册 Bean 等等 。

当然，这步结束后，Bean 并没有完成初始化，这里指的是 Bean 实例并未在这一步生成 。

```java
// AbstractApplicationContext.java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
   // 关闭旧的 BeanFactory (如果有)，创建新的 BeanFactory，加载 Bean 定义、注册 Bean 等等
   refreshBeanFactory();

   // 返回刚刚创建的 BeanFactory
   ConfigurableListableBeanFactory beanFactory = getBeanFactory();
   if (logger.isDebugEnabled()) {
      logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
   }
   return beanFactory;
}
```

AbstractRefreshableApplicationContext.java - 120

```java
protected final void refreshBeanFactory() throws BeansException {
     /**
        如果 ApplicationContext 中已经加载过 BeanFacotry，就销毁所有 Bean，关闭 BeanFactory
        注意，应用中 BeanFactory 本来就可以存在多个，这里可不是说应用全局是否有 BeanFactory ，而是当前
        ApplicationContext 是否有 BeanFactory 
     */
     if (this.hasBeanFactory()) {
          this.destroyBeans();
          this.closeBeanFactory();
     }

     try {
          // 初始化一个 DefaultListableBeanFactory，为什么用这个，我们马上说 。
          DefaultListableBeanFactory beanFactory = this.createBeanFactory();
          // 用于 BeanFactory 的序列化，我想部分人应该用不到。
          beanFactory.setSerializationId(this.getId());
          
          /**
             下面这两个方法很重要，别跟丢了，具体细节之后说
             设置 BeanFactory 的两个配置属性：是否允许 Bean 覆盖、是否允许循环引用
          */
          this.customizeBeanFactory(beanFactory);
          
          // 加载 Bean 到 BeanFactory 中
          this.loadBeanDefinitions(beanFactory);
          synchronized(this.beanFactoryMonitor) {
               this.beanFactory = beanFactory;
          }
     } catch (IOException var5) {
          throw new ApplicationContextException("I/O error parsing bean definition source for " + this.getDisplayName(), var5);
     }
}
```

看到这里的时候，我觉得读者就应站在高处看 ApplicationContext 了，ApplicationContext 继承自 BeanFactory，但是它不应该被理解为 BeanFactory 的实现类 ，而是说其内部持有一个实例化的 BeanFactory ( DefaultListableBeanFactory )。以后所有的 BeanFactory 相关的操作其实就是委托给这个实例来处理的 。

我们来说说为什么选择实例化 **DefaultListableBeanFactory** ？前面我们说了有个很重要的接口 ConfigurableListableBeanFactory，它实现了 BeanFactory 第二层的所有三个接口，我们重新过一遍继承图 ：

![2](SpringIOC图集/2.png)



我们可以看到 ConfigurableListableBeanFactory 只有一个实现类 DefaultListableBeanFactory，而且实现类 DefaultListableBeanFactory 还通过实现右边的 AbstractAutowireCapableBeanFactory 通吃了右路，结论就是，最底下这个家伙 DefaultListableBeanFactory 基本数是最牛的 BeanFactory 了，这也是为什么这里会使用这个类来实例化的原因 。

> 如果你想要在程序运行的时候动态往 Spring IOC 容器注册新的 bean，就会使用到这个类。那我们怎么在运行时获得这个实例呢？
>
> 之前我们说过 ApplicationContext 接口能获取到 AutowireCapableBeanFactory，就是最右上角那个，然后它向下转型就能得到 DefaultListableBeanFactory 了。
>
> 那怎么拿到 ApplicationContext 实例呢？如果你不会，说明你没用过 Spring。

继续往下之前，我们需要先了解 BeanDefinition ，





















