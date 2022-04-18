# Springboot

[TOC]

## 官网Overview

1. Create stand-alone Spring applications
2. Embed Tomcat, Jetty or Undertow directly (no need to deploy WAR files)
3. Provide opinionated 'starter' dependencies to simplify your build configuration
4. Automatically configure Spring and 3rd party libraries whenever possible
5. Provide production-ready features such as metrics, health checks and externalized configuration
6. Absolutely no code generation and no requirement for XML configuration

第二点中，在spring-boot-starter中已经集成了
第四点中，原先在springMVC中需要配置的applicationContext.xml(bean)、spring-mvc.xml(component-scan包路径，jsp prefix sufix)、web.xml(DisparcherServlet)都不再需要。靠@SpringBootApplication注解，等同于@Configuration、@AutoConfiguration、@ComponentScan。

- @EnableAutoConfiguration: enable Spring Boot’s auto-configuration mechanism
- @ComponentScan: enable @Component scan on the package where the application is located (see the best practices)
- @Configuration: allow to register extra beans in the context or import additional configuration classes（可以将bean成为配置bean）

依靠@Component可以将该类交给IOC容器管理


## 入口类

> 解析初始化SpringApplication.run(SplendidProApplication.class, args)

`return run(new Class[]{primarySource}, args);`
`return (new SpringApplication(primarySources)).run(args);`

1. 创建SpringApplication

```java
// 判断web应用的类型
this.webApplicationType = WebApplicationType.deduceFromClasspath();
// 设置初始化器 -->根据@EnableAutoConfiguration注解知道，ApplicationContextInitializer可以从spring.factories获取全路径
this.setInitializers(this.getSpringFactoriesInstances(ApplicationContextInitializer.class));
// 设置监听器 -->根据@EnableAutoConfiguration注解知道，ApplicationListener可以从spring.factories获取全路径
this.setListeners(this.getSpringFactoriesInstances(ApplicationListener.class));
```

2. run

```java
public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        ConfigurableApplicationContext context = null;
        Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList();
        this.configureHeadlessProperty();
        SpringApplicationRunListeners listeners = this.getRunListeners(args);
        listeners.starting();

        Collection exceptionReporters;
        try {
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            ConfigurableEnvironment environment = this.prepareEnvironment(listeners, applicationArguments);
            this.configureIgnoreBeanInfo(environment);
            Banner printedBanner = this.printBanner(environment);
            // Spring 上下文
            context = this.createApplicationContext();
            exceptionReporters = this.getSpringFactoriesInstances(SpringBootExceptionReporter.class, new Class[]{ConfigurableApplicationContext.class}, context);
            this.prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            // tomcat启动过程在这里
            // refreshContext -> refresh -> ((AbstractApplicationContext)applicationContext).refresh()(到spring了)
            //  -> onRefresh(ServletWebServerApplicationContext的)
            this.refreshContext(context);
            this.afterRefresh(context, applicationArguments);
            stopWatch.stop();
            if (this.logStartupInfo) {
                (new StartupInfoLogger(this.mainApplicationClass)).logStarted(this.getApplicationLog(), stopWatch);
            }

            listeners.started(context);
            this.callRunners(context, applicationArguments);
        } catch (Throwable var10) {
            this.handleRunFailure(context, var10, exceptionReporters, listeners);
            throw new IllegalStateException(var10);
        }

        try {
            listeners.running(context);
            return context;
        } catch (Throwable var9) {
            this.handleRunFailure(context, var9, exceptionReporters, (SpringApplicationRunListeners)null);
            throw new IllegalStateException(var9);
        }
    }

// 创建web容器
protected void onRefresh() {
    super.onRefresh();

    try {
        this.createWebServer();
    } catch (Throwable var2) {
        throw new ApplicationContextException("Unable to start web server", var2);
    }
}
```

## Starter

> 简化集成第三方的插件

原spring中需要做的事情

- 引入Jar
- 配置文件
- 向spring注入需要使用的类
- 测试代码

以tomcat-starter为例
- 可以配置核心设置，靠注解@PropertySource将application.propertise读入（~~是的吧~~） 

#### starter & autoconfigure
[官网参考](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.developing-auto-configuration.custom-starter)
A typical Spring Boot starter contains code to auto-configure and customize the infrastructure of a given technology, let’s call that "acme". To make it easily extensible, a number of configuration keys in a dedicated namespace can be exposed to the environment. Finally, a single "starter" dependency is provided to help users get started as easily as possible.

Concretely, a custom starter can contain the following:

- The autoconfigure module that contains the auto-configuration code for "acme".

- The starter module that provides a dependency to the autoconfigure module as well as "acme" and any additional dependencies that are typically useful. In a nutshell, adding the starter should provide everything needed to start using that library.

This separation in two modules is in no way necessary. If "acme" has several flavors, options or optional features, then it is better to separate the auto-configuration as you can clearly express the fact some features are optional. Besides, you have the ability to craft a starter that provides an opinion about those optional dependencies. At the same time, others can rely only on the autoconfigure module and craft their own starter with different opinions.

If the auto-configuration is relatively straightforward and does not have optional feature, merging the two modules in the starter is definitely an option.

以上：