http://junit.org/junit4/下载，跳转至GitHub，Junit有两个包junit.jar、hamcrest-core.jar

#### 使用：

在src目录下新建一个Test包，IntelliJ中需要在Project Stucture配置测试文件

1. 测试文件的结构应该与项目保持一致。
2. 测试类用Test作为后缀。
3. 测试方法用@Test注解，使用public void，不带参数，方法名前缀为test
4. 测试单元中的每个方法必须可独立测试
5. 编写完后右键选择需要测试的方法。

**注意：** 可能需要手工导入import static org.junit.Assert.*;包才能用assertEquals()

测试失败
1.失败：消息栈有提示预期值与实际值不一样
2.错误：消息栈会抛出异常并提示出错位置

#### Junit运行流程（以下是固定代码段）

流程：@BeforeClass -> @Before -> @Test -> @After -> @Before -> @Test -> @After -> @AfterClass

- @BeforeClass：该方法是静态的，内存中只存一份，适合加载配置文件
- @AfterClass：该方法也是静态的，通常对资源的清理
- @Before：test方法前后各执行一次
- @After：test方法前后各执行一次

#### 常用的注解

@Tset(expected = Exception.class, timeout=ms)
expected表示可能会有异常
timeout如果测试代码可能死循环，则在设定的时间退出测试方法

@Ignore
被此修饰的测试方法会被忽略

@RunWith
更改测试运行器（如Suite）

#### 测试套件

场景：有许多测试类等待测试
类上注解@RunWith(Suite.class) @Suite.SuiteClasses({TaskTest1.class,TaskTest2.class}) 即可
前者指定测试运行器，表示使用测试套件；后者指定有哪些测试类需要运行测试

#### 参数化设置

场景：测试同一个测试方法但是有一组数据等待测试

1. 类上注解@RunWith(Parameterized.class)
2. 声明变量存放预期值和结果值
3. 声明一个返回值为Collection的公共静态方法，并使用@Parameters进行修饰
4. 为测试类声明带有参数的公共构造函数，并为之赋值

#### 测试问题收集：

1.在测试的时候遇到报错：BeanNotOfRequiredTypeException: Bean named '' is expected to be of type '' but was actually of type 'com.sun.proxy.$Proxy25'
场景描述：在Spring的配置文件发现利用切面将这个报错的类织入了事务，应该是织入的反射报错了。
解决办法：去掉就没有问题了，具体解决办法不清楚，只能先注释掉