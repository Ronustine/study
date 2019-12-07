# 常用注解

提纲
[TOC]

## 引入：xml与注解
如果使用**xml文件**注入，那么xml文件应该是这样的：
```xml
<!-- 文件名为Beans.xml，放在resource文件夹下 -->

<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="car" class="com.ronustine.Car"></bean>
</beans>
```

同时，使用`ClassPathXmlApplicationContext`对象，并指定xml的位置（即从resource文件夹路径开始），如下：
```java
public static void main( String[] args ) {
    ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");
    System.out.println(ctx.getBean("person"));
}
```

如果是**注解**，那么看接下来的注解介绍，此时将使用`AnnotationConfigApplicationContext`对象。
`AnnotationConfigApplicationContext`需要使用一个类作为入口：如下面使用了自己写的`MainConfig`（称为配置类）
```java
public static void main( String[] args ) {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(MainConfig.class);
}
```

## ---- 注入注解介绍开始 ----

## @Configurantion & @Bean
```java
@Configuration
public class MainConfig {
    @Bean
    public Person person(){
        return new Person();
    }
}
```
注意：`@Bean`的使用，初始化容器后，`Person`对象的默认名称是方法名，若`@Bean(value="bean的名称")`那么`Person`的名称是指定的value，则从容器中读取Bean也应使用指定的value
```java
public static void main( String[] args ) {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(MainConfig.class);
    System.out.println(ctx.getBean("person"));
}
```

## @ComponentScan & @Component

### 简单使用
在配置类上写`@CompentScan`注解来开启扫描，出现在`basePackages`下的所有使用了`@componentScan`注解的类将会被注入容器。
```java
@ComponentScan(basePackages = {"com.xxx.testcompentscan"})
public class MainConfig {
}
```
### 排除用法 excludeFilters
排除@Controller注解的，同时排除某个具体类型XXXService
```java
@ComponentScan(basePackages = {"com.xxx.testcompentscan"},
    excludeFilters = {
        @ComponentScan.Filter(type = FilterType.ANNOTATION,value = {Controller.class}),
        @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE,value = {XXXService.class})
})
public class MainConfig {

}
```
###包含用法 includeFilters 
若使用包含的用法，需要把useDefaultFilters属性设置为false（true表示扫描全部的）
```java
@ComponentScan(basePackages = {"com.xxx.testcompentscan"},
includeFilters = {
    @ComponentScan.Filter(type = FilterType.ANNOTATION,value = {Controller.class, Service.class})
    },
useDefaultFilters = false)
public class MainConfig {
}
```

### @ComponentScan.Filter type的类型
```java
public enum FilterType {
    //注解形式 比如@Controller @Service @Repository @Compent 
    ANNOTATION,
    //指定的某个类
    ASSIGNABLE_TYPE,
    //aspectJ形式的，少用
    ASPECTJ,
    //正则表达式的，少用
    REGEX,
    //自定义的
    CUSTOM
}
```
`FilterType.ANNOTATION`, `FilterType.ASSIGNABLE_TYPE`上面已经使用过了，现在关注`FilterType.CUSTOM`的使用
```java
// 继承TypeFilter，实现match方法
public class SelfFilterType implements TypeFilter {
    @Override
    public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
        // 获取当前类的注解源信息
        AnnotationMetadata annotationMetadata = metadataReader.getAnnotationMetadata();
        // 获取当前类的class的源信息
        ClassMetadata classMetadata = metadataReader.getClassMetadata(); 
        // 获取当前类的资源信息
        Resource resource = metadataReader.getResource();
        if(classMetadata.getClassName().contains("dao")){
            return true;
        }
        return false; 
    }
}
// 使用SelfFilterType
@ComponentScan(basePackages = {"com.xxx.testcompentscan"},
    includeFilters = {
        @ComponentScan.Filter(type = FilterType.CUSTOM,value = SelfFilterType.class)
    },
    useDefaultFilters = false)
public class MainConfig {
}
```

## @Import
场景：自动装配原理，比如自动加载第三方jar包中的入口class

### 简单使用
导入组件的id为全类名路径：
注入后id不再是简单的person、car，而是`com.ronustine.testImport.Person`, `com.ronustine.testImport.Car`
```java
@Import(value = {Person.class, Car.class})
public class MainConfig {
}
```

### @Import & ImportSeletor
导入组件的id为全类名路径。
实现接口ImportSelector，覆盖方法selectImports，返回内容是类的全路径
```java
public class SelfImport implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        return new String[]{"com.ronustine.testimport.car"};
    }
}

@Import(value = {Person.class, SelfImport.class})
public class MainConfig {
}
```
**这里注入结果是Person和Car类**，id分别为：`com.ronustine.testimport.Person`, `com.ronustine.testimport.Car`
 
### @Import & ImportBeanDefinitionRegister
可以指定bean的名称
```java
public class SelfBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata annotationMetadata, BeanDefinitionRegistry beanDefinitionRegistry) {
        RootBeanDefinition beanDefinition = new RootBeanDefinition(Car.class);
        beanDefinitionRegistry.registerBeanDefinition("car", beanDefinition);
    }
}

@Import(value = {Person.class, SelfBeanDefinitionRegistrar.class})
public class MainConfig {
}
```
**这里注入结果是Person和Car类**，id分别为：`com.ronustine.testimport.Person`, `car`

## FactoryBean
场景：复杂的初始化，例子：SqlSessionFactoryBean
```java
public class PersonFactoryBean implements FactoryBean {
    // 实际上是获取这里返回的类
    @Override
    public Object getObject() throws Exception {
        return new Person();
    }

    // 类型
    @Override
    public Class<?> getObjectType() {
        return Person.class;
    }

    // 是否单例
    @Override
    public boolean isSingleton() {
        return true;
    }
}

@Configuration
public class MainConfig {
    @Bean
    public PersonFactoryBean personFactoryBean() {
        return new PersonFactoryBean();
    }
}

public class App {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(MainConfig.class);
        System.out.println(ctx.getBean("personFactoryBean"));
    }
}
```
**这里的ctx.getBean的结果是Person对象**
如果要获取PersonFactoryBean对象，那么这样获取：
`ctx.getBean("&personFactoryBean")`

## ---- 注入注解介绍结束 ----

## Bean的作用域
- Scope默认为单例的，容器在初始化的时候就加载。饿汉方式
- Scope指定为prototype时是多实例的（原型模式），在第一次使用才会加载。懒汉方式
（传送门：多实例下不能解决循环依赖的问题）

## Bean的懒加载
主要针对单实例的情况。容器在初始化不加载，第一次使用才加载
```java
@Bean
@Lazy
public Person person() {
    return new Person();
}
```

## @Conditional的条件判断
场景：有两个组件AAspect 和BLog，BLog组件是依赖于AAspect的组件
注意：Bean的加载顺序，使用@Configuration & @Bean控制顺序

```java
// 使用：自己创建一个SelfCondition的类 实现Condition接口
public class SelfCondition implements Condition {
    /** 
    *
    * @param context
    * @param metadata 
    * @return
    */
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        //判断容器中是否有AAspect的组件
        if(context.getBeanFactory().containsBean("aaspect")) {
            return true;
        }
        return false; 
    }
}

// 配置类使用SelfCondition
public class MainConfig {
    @Bean
    public AAspect aaspect() {
        return new AAspect();
    }
    //当切 容器中有AAspect的组件，那么BLog才会被实例化. @Bean
    @Conditional(value = SelfCondition.class)
    public BLog bLog() {
        return new BLog();
    }
}

```

## Bean的生命周期
bean的创建 -> 初始化 -> 销毁方法，是由IOC通过回调来控制

在配置Bean的时候可以设置initMethod、destroyMethod
```java
@Configuration
public class MainConfig {
    @Bean(initMethod = "init", destroyMethod = "destroy")
    public Person person() {
        return new Person();
    }
}

public class Person {
    public Person () {
        System.out.println("person--constructor");
    }
    public void init() {
        System.out.println("person--init");
    }
    public void destroy() {
        System.out.println("person--destroy");
    }
}

public class App {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(MainConfig.class);
        System.out.println(ctx.getBean("person"));
        ctx.close();
    }
}

/*
输出：
person--constructor
person--init
com.ronustine.testlifecycle.Person@24b1d79b
person--destroy
*/
```
注意：
- 在单实例中，容器初始化时init会被调用，在容器销毁的时候destroy会被调用
- 在多实例中，在bean被使用的时候init才会被调用，销毁则交由JVM的垃圾收集

现在，添加`InitializingBean`，`DisposableBean`的两个接口实现bean的初始化以及销毁方法。（InitializingBean可用于修改属性，多数据源）

现在，根据JSR250规范 增加提供的注解@PostConstruct 和@ProDestory标注的方法

现在，添加`BeanBeanPostProcessor`，后置处理器。在Bean的初始化方法（init）前后调用。可用于修改Bean的属性
```java
// 配置类，注入Person，SelfBeanPostProcessor
@Configuration
public class MainConfig {
    @Bean(initMethod = "init", destroyMethod = "destroyMethod")
    public Person person() {
        return new Person();
    }
    @Bean
    public SelfBeanPostProcessor beanPostProcessor() {
        return new SelfBeanPostProcessor();
    }
}

// 注入的类
public class Person implements InitializingBean, DisposableBean {
    public Person () {
        System.out.println("person--constructor");
    }
    @PostConstruct
    public void postConstuct() {
        System.out.println("person--postConstuct");
    }
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("person--propertiesSet");
    }
    public void init() {
        System.out.println("person--init");
    }
    @PreDestroy
    public void preDestroy() {
        System.out.println("person--preDestroy");
    }
    @Override
    public void destroy() {
        System.out.println("person--destroy");
    }
    public void destroyMethod() {
        System.out.println("person--destroyMethod");
    }
}

// 注入 用于初始化方法前后增加方法 的Bean
public class SelfBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object o, String s) throws BeansException {
        System.out.println(s + "--beanPostProcessor--before");
        return o;
    }
    @Override
    public Object postProcessAfterInitialization(Object o, String s) throws BeansException {
        System.out.println(s + "--beanPostProcessor--after");
        return o;
    }
}
/*
输出：
person--constructor
person--beanPostProcessor--before
person--postConstuct
person--propertiesSet
person--init
person--beanPostProcessor--after

person--preDestroy
person--destroy
person--destroyMethod
*/
```
TODO 比较模糊，等源码过完再完善这里

`InitializingBean`：静态检查或者属性的赋值

## @Value & @PropertySource
给组件赋值，**PropertySource必须写在配置类上**
```java
@Configuration
@PropertySource(value = "person.properties")
public class MainConfig {
    @Bean
    public Person person() {
        return new Person();
    }
}

public class Person {
    @Value("${person.age}")
    private Integer age;

    public void setAge(Integer age){
        this.age = age;
    }
    public Integer getAge() {
        return this.age;
    }
    @Override
    public String toString() {
        return "Person{" +
                "age=" + age +
                '}';
    }
}

// person.properties文件内容为  person.age=2222
```
注意：`@ImportResource`是导入xml的

## 自动装配

### @Autowired
- 默认ByName
- 可以放在成员变量上
- 可以放在方法上：set方法、构造方法、配置类上的入参中（可不写）

```java
@Component
public class Person {
    private Integer age;
    public void setAge(Integer age){
        this.age = age;
    }
    public Integer getAge() {
        return this.age;
    }
    @Override
    public String toString() {
        return "Person{" +
                "age=" + age +
                '}';
    }
}

@Controller
public class PersonController {

    //@Qualifier(value = "person2")
    @Autowired
    Person person;
    public Person getPerson() {
        return person;
    }
    public void setPerson(Person person) {
        this.person = person;
    }
    @Override
    public String toString() {
        return "PersonController{" +
                "person=" + person +
                '}';
    }
}

// 这里有两个Person的实例，分别是person、person2
@Configuration
public class MainConfig {
    @Bean
    public Person person() {
        Person p = new Person();
        p.setAge(44445);
        return p;
    }
    @Bean
    public Person person2() {
        return new Person();
    }
}

public class App {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(MainConfig.class);
        System.out.println(ctx.getBean("personController"));
    }
}
```

即PersonController中的成员变量是依据名称在容器中查找Bean的，如果非要加载其他实例类进这个成员变量（如person2），那么`@Autowired`上面再加一个`@Qualifier("person2")`，就能放进去了

### @Autowired(required = false)
可以不强制装配，找不到就不装配，不会报错

### @Resource(JSR250规范) 
功能和`@AutoWired`的功能差不多一样，但是不支持`@Primary`和`@Qualifier`的支持

### @InJect(JSR330规范)
需要导入jar包依赖
功能和支持`@Primary`功能 ,但是没有`Require=false`的功能

## @Profile
根据环境来标志不同的Bean
- 没有标志为@Profile的bean不管在什么环境都可以被激活
- 标识在类上，那么只有当前环境匹配，整个配置类才会生效
- 标识在Bean上 ，那么只有当前环境的Bean才会被激活

激活切换环境的方法
方法一:通过运行时jvm参数来切换 -Dspring.profiles.active=test|dev|prod 
方法二:通过代码的方式来激活
```java
 public static void main(String[] args) {
    AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(); 
    ctx.getEnvironment().setActiveProfiles("test","dev");
    ctx.register(MyConfig.class); 
    ctx.refresh(); 
    printBeanName(ctx);
}
```