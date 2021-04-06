# IOC

[TOC]

> 核心：资源集中管理，实现可配置并且易于管理；降低资源之间的耦合度。
> 此文章解析AnnotationConfigApplicationContext初始化。父类是GenericApplicationContext。父类的父类，即起点，是AbstractApplicationContext。

## 准备：AbstractApplicationContext
核心类AbstractApplicationContext，特别关注refresh()。下面给出此类的官方介绍，从整体把握容器。
```java
/**
 * 抽象类，简单实现了有共性的基础功能。使用了模板方法设计模式，交由子类实现抽象方法。
 * Abstract implementation of the {@link org.springframework.context.ApplicationContext}
 * interface. Doesn't mandate the type of storage used for configuration; simply
 * implements common context functionality. Uses the Template Method design pattern,
 * requiring concrete subclasses to implement abstract methods.
 *
 * 区别于BeanFactory，ApplicationContext需要检测在beanFactory定义的特殊bean，然后自动注册BeanFactoryPostProcessors，BeanPostProcessors，ApplicationListeners。
 * <p>In contrast to a plain BeanFactory, an ApplicationContext is supposed
 * to detect special beans defined in its internal bean factory:
 * Therefore, this class automatically registers
 * {@link org.springframework.beans.factory.config.BeanFactoryPostProcessor BeanFactoryPostProcessors},
 * {@link org.springframework.beans.factory.config.BeanPostProcessor BeanPostProcessors},
 * and {@link org.springframework.context.ApplicationListener ApplicationListeners}
 * which are defined as beans in the context.
 *
 * 会提供MessageSource做国际化。此外，提供默认实现的multicaster广播器。
 * <p>A {@link org.springframework.context.MessageSource} may also be supplied
 * as a bean in the context, with the name "messageSource"; otherwise, message
 * resolution is delegated to the parent context. Furthermore, a multicaster
 * for application events can be supplied as an "applicationEventMulticaster" bean
 * of type {@link org.springframework.context.event.ApplicationEventMulticaster}
 * in the context; otherwise, a default multicaster of type
 * {@link org.springframework.context.event.SimpleApplicationEventMulticaster} will be used.
 *
 * 靠继承的DefaultResourceLoader来加载。除非重写方法getResourceByPath，否则默认按类路径找。
 * <p>Implements resource loading by extending
 * {@link org.springframework.core.io.DefaultResourceLoader}.
 * Consequently treats non-URL resource paths as class path resources
 * (supporting full class path resource names that include the package path,
 * e.g. "mypackage/myresource.dat"), unless the {@link #getResourceByPath}
 * method is overridden in a subclass.
 *
 * @see #refreshBeanFactory
 * @see #getBeanFactory
 * @see org.springframework.beans.factory.config.BeanFactoryPostProcessor
 * @see org.springframework.beans.factory.config.BeanPostProcessor
 * @see org.springframework.context.event.ApplicationEventMulticaster
 * @see org.springframework.context.ApplicationListener
 * @see org.springframework.context.MessageSource
 */
public abstract class AbstractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext, DisposableBean {
        ...
}
```
GenericApplicationContext，这里会使用默认的BeanFactory-DefaultListableBeanFactory
```java
/**
 * 用DefaultListableBeanFactory作为BeanFactory。为了配合bean definition readers，还实现了BeanDefinitionRegistry。
 * Generic ApplicationContext implementation that holds a single internal
 * {@link org.springframework.beans.factory.support.DefaultListableBeanFactory}
 * instance and does not assume a specific bean definition format. Implements
 * the {@link org.springframework.beans.factory.support.BeanDefinitionRegistry}
 * interface in order to allow for applying any bean definition readers to it.
 *
 * 典型用法：通过BeanDefinitionRegistry接口，注册各种bean definitions。（此类偷懒，直接调用自家BeanFactory的）
 * <p>Typical usage is to register a variety of bean definitions via the
 * {@link org.springframework.beans.factory.support.BeanDefinitionRegistry}
 * interface and then call {@link #refresh()} to initialize those beans
 * with application context semantics (handling
 * {@link org.springframework.context.ApplicationContextAware}, auto-detecting
 * {@link org.springframework.beans.factory.config.BeanFactoryPostProcessor BeanFactoryPostProcessors},
 * etc).
 *
 * 区别于其他ApplicationContext，每次refresh都会创建新的BeanFactory。这个只会在refresh()调用一次。
 * <p>In contrast to other ApplicationContext implementations that create a new
 * internal BeanFactory instance for each refresh, the internal BeanFactory of
 * this context is available right from the start, to be able to register bean
 * definitions on it. {@link #refresh()} may only be called once.
 *
 * <p>Usage example:
 *
 * <pre class="code">
 * GenericApplicationContext ctx = new GenericApplicationContext();
 * XmlBeanDefinitionReader xmlReader = new XmlBeanDefinitionReader(ctx);
 * xmlReader.loadBeanDefinitions(new ClassPathResource("applicationContext.xml"));
 * PropertiesBeanDefinitionReader propReader = new PropertiesBeanDefinitionReader(ctx);
 * propReader.loadBeanDefinitions(new ClassPathResource("otherBeans.properties"));
 * ctx.refresh();
 *
 * MyBean myBean = (MyBean) ctx.getBean("myBean");
 * ...</pre>
 *
 * For the typical case of XML bean definitions, simply use
 * {@link ClassPathXmlApplicationContext} or {@link FileSystemXmlApplicationContext},
 * which are easier to set up - but less flexible, since you can just use standard
 * resource locations for XML bean definitions, rather than mixing arbitrary bean
 * definition formats. The equivalent in a web environment is
 * {@link org.springframework.web.context.support.XmlWebApplicationContext}.
 *
 * <p>For custom application context implementations that are supposed to read
 * special bean definition formats in a refreshable manner, consider deriving
 * from the {@link AbstractRefreshableApplicationContext} base class.
 *
 * @since 1.1.2
 * @see #registerBeanDefinition
 * @see #refresh()
 * @see org.springframework.beans.factory.xml.XmlBeanDefinitionReader
 * @see org.springframework.beans.factory.support.PropertiesBeanDefinitionReader
 */
public class GenericApplicationContext extends AbstractApplicationContext implements BeanDefinitionRegistry {
    ...
}
```
AnnotationConfigApplicationContext，有reader、scaner可以在构造器传入需要注册的类，比如@Configuration
```java
/**
 * Standalone application context, accepting annotated classes as input - in particular
 * {@link Configuration @Configuration}-annotated classes, but also plain
 * {@link org.springframework.stereotype.Component @Component} types and JSR-330 compliant
 * classes using {@code javax.inject} annotations. Allows for registering classes one by
 * one using {@link #register(Class...)} as well as for classpath scanning using
 * {@link #scan(String...)}.
 *
 * <p>In case of multiple {@code @Configuration} classes, @{@link Bean} methods defined in
 * later classes will override those defined in earlier classes. This can be leveraged to
 * deliberately override certain bean definitions via an extra {@code @Configuration}
 * class.
 *
 * <p>See @{@link Configuration}'s javadoc for usage examples.
 *
 * @see #register
 * @see #scan
 * @see AnnotatedBeanDefinitionReader
 * @see ClassPathBeanDefinitionScanner
 * @see org.springframework.context.support.GenericXmlApplicationContext
 */
public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry {
    ...
}
```

## 准备：BeanDefinition

是Spring BeanFactory中Bean的描述，一个 BeanDefinition 描述了一个 Bean 实例，实例包含属性值、构造方法参数值以及更多实现信息

## 准备：ApplicationListener

监听器，注册到多播器中`ApplicationEventMulticaster`，这里使用了观察者模式。

#### 用法

发布方式，见下
1. 实现`ApplicationListener`接口，可注册自己的监听器
2. `AnnotationConfigApplicationContext` `publishEvent`发布事件，事件可以继承`ApplicationContextEvent`
```java
// 建立自己的监听器
@Component
public class SelfApplicationListener implements ApplicationListener {
    @Override
    public void onApplicationEvent(ApplicationEvent applicationEvent) {
        System.out.println("通过容器发布了一个事件" + applicationEvent.toString());
    }
}

// 发布事件
public class App {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(MainConfig.class);
        ctx.publishEvent(new ApplicationEvent("手动发布事件") {
            @Override
            public Object getSource() {
                return super.getSource();
            }
        });
    }
}

/*
输出：
通过容器发布了一个事件org.springframework.context.event.ContextRefreshedEvent[source=org.springframework.context.annotation.AnnotationConfigApplicationContext@31cefde0: startup date [Tue Jun 25 22:55:17 CST 2019]; root of context hierarchy]
通过容器发布了一个事件com.ronustine.testapplicationlistener.App$1[source=手动发布事件]
*/
```
使用默认的并不会有异步线程处理，需要自己写一个，@Component命名为`applicationEventMultiCaster`，并继承SimpleApplicationEventMultiCaster。设置线程池

#### 源码

位置：`AnnotationConfigApplicationContext`初始化 -> `refresh()` -> `initApplicationEventMulticaster()`
- 使用提供的多播器
- 没有则使用默认的多播器`SimpleApplicationEventMulticaster`，可以异步，但缺少线程池，需要自己 `setTaskExecutor()`

```java
@Component(value = "applicationEventMulticaster")
public class SelfMulticaster extends SimpleApplicationEventMulticaster{
    public SelfMulticaster () {
        setTaskExecutor(Executors.newSingleThreadExecutor());
    }

}
```

## 准备：BeanFactoryPostProcessor

Bean工厂的后置处理器。当Bean定义加载到容器中的时候会调用，此时还未实例化Bean。可以利用这个空隙修改BeanDefinition的相关内容。不常用。

#### 用法

```java
@Component
public class SelfBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory configurableListableBeanFactory) throws BeansException {
        System.out.println("调用了SelfBeanFactoryPostProcessor postProcessBeanFactory");

        for (String name: configurableListableBeanFactory.getBeanDefinitionNames()){
            if ("person".equals(name)) {
                System.out.println("这里有" + name);
                // 这里可以修改BeanDefinition相关内容
                BeanDefinition beanDefinition = configurableListableBeanFactory.getBeanDefinition(name);
                beanDefinition.setLazyInit(true);
            }
        }
    }
}

@ComponentScan(basePackages = "com.ronustine.testbeanfactorypostprocessor")
public class MainConfig {
}

@Component
public class Person {
    public Person(){
        System.out.println("初始化");
    }
}

/*
输出
调用了SelfBeanFactoryPostProcessor postProcessBeanFactory
这里有person
初始化
*/
```

#### 例子：BeanDefinitionRegistryPostProcessor​

所有的bean定义信息将要被加载到容器中，Bean实例还没有被初始化。继承自BeanFactoryPostProcessor，在解析成BeanDefinition之前的时候
```java
@Component
public class SelfBeanDefinationRegisterPostProcessor implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        System.out.println("SelfBeanDefinationRegisterPostProcessor的postProcessBeanDefinitionRegistry方法");
        System.out.println("bean定义的数据量:"+registry.getBeanDefinitionCount());
        RootBeanDefinition rootBeanDefinition = new RootBeanDefinition(SelfLog.class);
        registry.registerBeanDefinition("SelfLog",rootBeanDefinition);
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("SelfBeanDefinationRegisterPostProcessor的postProcessBeanFactory方法");
        System.out.println(beanFactory.getBeanDefinitionCount());
    }
}
```

## 正式：AnnotationConfigApplicationContext

> BeanFactory与ApplicationContext
![1](3-1.jpg)

这个对象做了什么操作？
`AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(MainConfig.class);`

AnnotationConfigApplicationContext继承自GenericApplicationContext，此Generic..有一个BeanFactory: DefaultListableBeanFactory
```java
// 这里开始
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
    // 转1，初始化了 注解Bean定义读取器 & 扫描器
    this();

    // 注册传入的配置类annotatedClasses
    this.register(annotatedClasses);

    // 转2，重点
    this.refresh();
}

// 1. 接this();
public AnnotationConfigApplicationContext() {
    // 注解Bean类的读取器，设置@Condition的注解解析器、后置处理器
    // 重点在 -> AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
    // 里面还有这个：注册注解配置解析器（都是Spring内部需要使用的组件、解析器），把有用的都添加到BeanDefinitionMap中了（仅仅是BeanDefinition，还未实例）
	this.reader = new AnnotatedBeanDefinitionReader(this);

    // 类路径Bean的扫描器，使用默认的过滤器（增加对@Component的解析支持）
    // 重点在 -> registerDefaultFilters();
	this.scanner = new ClassPathBeanDefinitionScanner(this);
}

// 2. 接refresh。
// 该方法在AbstractApplicationContext中
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // 初始化多播器，有默认实现。可以自己实现一个。
            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // springboot使用的
            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // 注册监听器，并发布早期事件（多播器还没初始化的时候攒起来的事件，onRefresh中注册的也是）
            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
        } catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        } finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```

#### invokeBeanFactoryPostProcessors
```java
// BeanFactory分两类来处理，一类是继承BeanDefinitionRegistry，一类不是。第一类会专门多找BeanDefinitionRegistryPostProcessor这种类的处理器。（AnnotationConfigApplicationContext是继承自GenricApplicationContext, 其中的BeanFactory是DefaultListableBeanFactory，继承自BeanDefinitionRegistry）
// 接refresh().invokeBeanFactoryPostProcessors()。调用PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors
public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
    ...
    if (beanFactory instanceof BeanDefinitionRegistry) {
        // 说明：分两种PostProcess，BeanFactoryPostProcessor和BeanDefinitionRegistryPostProcessor。 这里称作[regularPostProcessor]、[registryProcessor]。他们的关系是registryProcessor继承自regularPostProcessor，扩展了一个方法。
        // 说明：这里大多是处理registryProcessor。入参与容器（BeanFactory）的registryProcessor按下面的优先级分步执行方法invokeBeanDefinitionRegistryPostProcessors(...),即postProcessBeanDefinitionRegistry();
        // 0. 取方法入参中的registryProcessor，执行；（实际上，refresh()进来的没有）
        // 1. 取容器PriorityOrdered注解标注的registryProcessor，排序 + 执行；
        // 2. 取容器Ordered注解标注的registryProcessor，排序 + 执行；
        // 3. 取容器剩下的registryProcessor，排序 + 执行；
        // 分别执行前面0～3步骤出现过的registryProcessor 和入参的regularPostProcessor，执行方法postProcessBeanFactory();
    } else {
        // 无特殊处理，直接调用入参的
        invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
    }

    // 说明：处理容器中的BeanFactoryPostProcessor，按以下优先级
    // 0. 取容器PriorityOrdered注解标注的，排序 + 执行
    // 1. 取容器Ordered注解标注的，排序 + 执行
    // 2. 其余的，排序 + 执行

    // 最后，主要逻辑：先BeanDefinitionRegistryPostProcessor，后BeanFactoryPostProcessor
}
```
方法`postProcessBeanDefinitionRegistry(...)`
特别解析这个RegistryPostProcessor: ConfigrationClassPostProcessor（没别的了，只有这个）
作用：筛出注解@Configuration相关的配置类，解析
```java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    ...
    processConfigBeanDefinitions(...);
    ...
}

public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    // 说明
    // 1. 取出自己注解@Configuration相关的配置类,称作[configCandidates]，没有则返回。有则排序并继续
    // 2. 取名器[BeanNameGenerator]、解析器[ConfigurationClassParser]
    // 3. 新建对象ConfigurationClassParser（给出解析过程），依次解析configCandidates
    do{
        // 依次解析configCandidates

        // 入口是解析自己的@Configuration，主要内容：
        // 1. 转processConfigurationClass()，此方法有多处递归调用解析
        parser.parse();
        ...
        // 装载BeanDefinetion
        this.reader.loadBeanDefinitions(configClasses);

    } while (!configCandidates.isEmpty())

    ...
}

// 类ConfigurationClassParser
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
    ...
    // 会递归
    SourceClass sourceClass = asSourceClass(configClass);
    do {
        // 转doProcessConfigurationClass()，成员变量（递），@ComponentScan（递），@Import（递）， @ImportResource（递），@Bean（递），找superclass
        // 只是解析，做准备工作
		sourceClass = doProcessConfigurationClass(configClass, sourceClass);
    } while (sourceClass != null);

    this.configurationClasses.put(configClass, configClass);
}
```

## BeanPostProcessor
> Bean后置处理器，它是Spring中定义的接口，在Spring容器的创建过程中（具体为Bean初始化前后）会回调BeanPostProcessor中定义的两个方法
```java
public interface BeanPostProcessor {

    // postProcessBeforeInitialization方法会在每一个bean对象的初始化方法调用之前回调
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    
    // postProcessAfterInitialization方法会在每个bean对象的初始化方法调用之后被回调
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

}
```