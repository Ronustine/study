[TOC]

## 类加载过程
**不是一次性加载完成的，用到才会加载**。加载到使用的过程：加载 -> 验证 -> 准备 -> 解析 -> 初始化 -> 使用 -> 卸载。
- 加载：在硬盘上查找并通过IO读入字节码文件，使用到类时才会加载，例如调用类的main()方法，new对象等等；
- 验证：校验字节码文件的正确性；
- 准备：给类的静态变量分配内存，并赋予默认值；
- 解析：将符号引用替换为直接引用，该阶段会把一些静态方法(符号引用，比如 main()方法)替换为指向数据所存内存的指针或句柄等(直接引用)，这是所谓的静态链接过程(类加载期间完成)，动态链接是在程序运行期间完成的将符号引用替换为直接引用；
- 初始化：对类的静态变量初始化为指定的值，执行静态代码块；

## 加载规则
- 对类的初始化会对父类做初始化，但对接口不会；
- 数组中的类不会被类加载器加载；


## 类加载器
![1](img/JVM_1.jpg)
上面加载过程主要是通过加载器来实现的，java有
- Boostrap类加载器：负责加载支撑JVM运行的位于JRE的lib目录下的核心类库，比如 rt.jar、charsets.jar等；
- Extension类加载器：负责加载支撑JVM运行的位于JRE的lib目录下的ext扩展目录中的JAR类包；
- App类加载器：负责加载ClassPath路径下的类包，主要就是加载你自己写的那些类自定义加载器：负责加载用户自定义路径下的类包；

可以通过添加参数查看加载了哪些类：`-verbose:class`

#### 自定义一个类加载器
自定义类加载器只需要继承java.lang.ClassLoader类，该类有两个核心方法，一个是 loadClass(String, boolean)，实现了双亲委派机制，大体逻辑 ：
1. 首先，检查一下指定名称的类是否已经加载过，如果加载过了，就不需要再加载， 直接返回；
2. 如果此类没有加载过，那么，再判断一下是否有父加载器；如果有父加载器，则由 父加载器加载（即调用parent.loadClass(name, false);）.或者是调用bootstrap类加 载器来加载；
3. 如果父加载器及bootstrap类加载器都没有找到指定的类，那么调用当前类加载器 的findClass方法来完成类加载。

还有一个方法是findClass，默认实现是抛出异常，所以我们自定义类加载器主要是重写 findClass方法。

#### 双亲委派
加载某个类时会先委托父加载器寻找目标类，找不 到再委托上层父加载器加载，如果所有父加载器在自己的加载类路径下都找不到目标类，则 在自己的类加载路径中查找并载入目标类。

缘由：
- 沙箱安全机制：自己写的java.lang.String类不会被加载，这样便可以防止 核心API库被随意篡改
- 避免类的重复加载：当父亲已经加载了该类时，就没有必要子ClassLoader再加载一次，保证被加载类的唯一性

双亲委派的关键代码
```java
// 位置：jre/lib/rt.jar!/java/lang/ClassLoader.class
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                // 如果有，先给双亲load
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                // 双亲没找到，自己再找
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}
```

#### 打破双亲委派机制
重写ClassLoader的loadClass。
###### Tomcat为例，为什么要打破？
1. 一个web容器可能需要部署两个应用程序，不同的应用程序可能会依赖同一个第三方类库的不同版本，要保证每个应用程序的类库都是独立的，保证相互隔离；
2. 部署在同一个web容器中相同的类库相同的版本可以共享。否则，如果服务器有n个应用程序，那么要有n份相同的类库加载进虚拟机；
3. web容器也有自己依赖的类库，不能与应用程序的类库混淆。基于安全考虑，应该让容器的类库和程序的类库隔离开来；
4. web容器要支持jsp的修改，我们知道，jsp 文件最终也是要编译成class文件才能在虚拟 机中运行，但程序运行后修改jsp已经是司空见惯的事情， web容器需要支持jsp修改后不用重启。

Tomcat自定义加载器功能
- commonLoader：Tomcat最基本的类加载器，加载路径中的class可以被 Tomcat容器本身以及各个Webapp访问；
- catalinaLoader：Tomcat容器私有的类加载器，加载路径中的class对于 Webapp不可见；
- sharedLoader：各个Webapp共享的类加载器，加载路径中的class对于所有 Webapp可见，但是对于Tomcat容器不可见； 
- WebappClassLoader：各个Webapp私有的类加载器，加载路径中的class只对 当前Webapp可见；

CommonClassLoader能加载的类都可以被CatalinaClassLoader和SharedClassLoader使用，从而实现了公有类库的共用，而CatalinaClassLoader和SharedClassLoader自己能加载的类则与对方相互隔离。 
WebAppClassLoader可以使用SharedClassLoader加载到的类，但各个 WebAppClassLoader实例之间相互隔离。
而JasperLoader的加载范围仅仅是这个JSP文件所编译出来的那一个.Class文件，它出现的目的就是为了被丢弃：当Web容器检测到JSP文件被修改时，会替换掉目前的 JasperLoader的实例，并通过再建立一个新的Jsp类加载器来实现JSP文件的热加载功能。

tomcat为了实现隔离性，没有遵守双亲委派这个约定，每个webappClassLoader加载自己的目录下的class文件，不会传递给父类加载器，打破了双亲委派机制。

###### SPI（待补充）
`Class.forName("com.mysql.jdbc.Driver")`
Launcher ServerLoader 改变了上下文加载器

#### JClasslib插件
> idea插件，比javap -v清晰

view -> show bytecode with jclasslib

## JVM 运行时数据区
- 堆
- 栈
- 方法区
- 本地方法栈
- 程序计数器

#### 栈
当开启了线程运行后，会给这个线程分配栈，存放运行时涉及到的方法都会压入栈，每个方法都会成为栈中的栈帧，栈帧存放局部变量等。
```java
public class Math{
    public int compute() {
        int a = 1;
        int b = 2;
        int c = (a + b) * 10;
        return c;
    }
    public static void main(String[] args) {
        Math math = new Math();
        math.compute();
    }
}
```
如上，线程开始执行后，执行到哪个方法则压入则把那个方法做成栈帧压入栈，栈帧有该方法自己的变量，操作，直到执行完方法才会出栈。
可以将class文件反汇编成更可读的内容分析：`javap -c、javap -v`(另需参考资料：JVM指令手册)

- 局部变量表：存放变量的地方，如果是对象则保存引用；
- 操作数栈：JVM把操作数栈作为它的工作区——大多数指令都要从这里弹出数据，执行运算，然后把结果压回操作数栈。在操作数栈中存储数据的方式和在局部变量区中是一样的：如int、long、float、double、reference和returnType的存储。对于byte、short以及char类型的值在压入到操作数栈之前，也会被转换为int；
- 方法出口：方法结束后返回到上一个调用方法的位置
- 动态链接：与类加载过程的解析-静态链接对应。是在运行时将符号引用转化成直接引用的过程，即`math.compute();`

相关参数：`-Xss1024K`表示栈的大小，直接决定最深调用能压入多少方法，值越大，能开启线程的数量就越小，因为栈的总大小是固定的

#### 程序计数器
字节码执行引擎操作的，记录当前线程正在执行的jvm指令码那一行的行号

#### 方法区（元空间）
> 1.8之后为元空间，使用直接内存，不使用分配给JVM的内存。

类加载子系统加载的，包含常量池、静态变量、类元信息
相关参数：`-XX:MetaspaceSize、-XX:MaxMetaspaceSize`，可以限制对直接内存的无限索取

#### 本地方法栈
native修饰的是本地方法，是以前调用C语言库的解决方案，现在调用C语言库就有很多方案，如thrift框架

**注意**：栈、程序计数器、本地方法栈都是线程私有的

#### 堆
每个对象的对象头会有指向方法区的指针。
堆区分年轻代、老年代。其中年轻代有Eden、Suvivor（from、to）区
相关参数： `-Xms`堆最小 `-Xmx`堆最大 `-Xmn`新生代大小

## GC
> 考虑三件事情：
> - 哪些内存需要回收
> - 什么时候回收
> - 如何回收

Eden满了会触发minor gc，对Eden、Suvivor区根据可达性分析回收不再使用的对象
老年代满了会触发full gc
如果回收后还是没有内存，就会outOfMemory异常。

面对的实际问题：
上线前预估机器的容量（常见的电商，普通的4核8G足够了）。在业务系统中，需要根据业务代码估算出每秒并发量以及生成出来对象的大小，对象在堆中存在的时间（根据业务代码处理的时长决定）

#### 哪些需要回收
###### 引用计数算法
只要对象有被引用，就记一次，计数器为零的对象就是不可能再被使用的。但是这个算法很难解决面对两个对象的相互引用的情况。JVM不会使用这个。

###### 可达性分析算法
主流的内存管理是这个算法判定的。通过一系列的称为 “GC Roots” 的对象作为起点，从这些节点开始向下搜索，找到的对象都标记为非垃圾对象，其余未标记的对象都是垃圾对象。

GC Roots根节点：
- 线程栈的本地变量；
- 本地方法栈的变量、静态变量；
- Java虚拟机内部的引用，如基本数据类型对应的Class对象，一些常驻的异常对象（比如NullPointExcepiton、OutOfMemoryError）等，还有系统类加载器；
- 所有被同步锁（synchronized关键字）持有的对象；

###### 引用类型相关
> 引用类型一般分为四种：强引用、软引用、弱引用、虚引用

强引用：普通的变量引用；

软引用：将对象用SoftReference软引用类型的对象包裹，正常情况不会被回收，但是GC做完后发现释放不出空间存放新的对象，则会把这些软引用的对象回收掉（利用类似钩子的方法回收，不会再次GC的）。软引用可用来实现内存敏感的高速缓存；
`public static SoftReference<User> user = new SoftReference<User>(new User());`

弱引用：将对象用WeakReference软引用类型的对象包裹，只能生存到下一次垃圾收集发生为止，很少用；
`public static WeakReference<User> user = new WeakReference<User>(new User());`

虚引用：虚引用也称为幽灵引用或者幻影引用，它是最弱的一种引用关系，几乎不用。为一个对象设置虚引用关联的唯一目的只是为了能在这个对象被收集器回收时收到一个系统通知；

###### ~~finalize() 不推荐，忘了吧~~
> 是Java刚诞生时为了使传统C、C++程序员更容易接受Java所做出的一项妥协。它的运行代价高昂，不确定性大，无法保证各个对象的调用顺序，如今已被官方明确声明为不推荐使用的语法。

即使在可达性分析算法中判定为不可达的对象，也不是“非死不可”的，这时候它们暂时还处于“缓刑”阶段，要真正宣告一个对象死亡，至少要经历两次标记过程：如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链，那它将会被第一次标记，随后进行一次筛选，筛选的条件是此对象是否有必要执行finalize()方法。假如对象没有覆盖finalize()方法，或者finalize()方法已经被虚拟机调用过一次，那么虚拟机将直接回收。

###### 无用的类
方法区主要回收的是无用的类，那么如何判断一个类是无用的类的呢？ 类需要同时满足下面3个条件才能算是 “无用的类” ：
- 该类所有的实例都已经被回收，也就是 Java 堆中不存在该类的任何实例；
- 加载该类的 ClassLoader 已经被回收；
- 该类对应的 java.lang.Class 对象没有在任何地方被引用，无法在任何 地方通过反射访问该类的方法。

###### 逃逸分析（参数）
JVM可以根据逃逸分析判断对象的作用域，来决定对象分配到栈还是堆中。

逃逸分析：即在JVM即时编译的时候对变量的作用域进行分析，如果作用域在一个方法中用完便不再使用（比如不会返回给上一个方法），那么就会在这个方法所在的栈帧中给这个对象分配空间使用。这样可以减少堆的压力。

JVM的运行模式（在解析的时候对代码做一些优化，逃逸分析就在其中）
- 解释模式：使用解释器（-Xint 强制JVM使用解释模式），执行一行JVM字节码就编译一行为机器码；
- 编译模式：只使用编译器（-Xcomp JVM使用编译模式），先将所有JVM字节码一次编译为机器码，然后一次性执行所有机器码；
- 混合模式：依然使用解释模式执行代码，但是对于一些 "热点" 代码采用编译模式执行，JVM一般采用混合模式执行代码（这些热点代码对应的机器码会被缓存起来，下次再执行无需再编译，这就是我们常见的JIT Just In Time Compiler即时编译技术）；

JVM对于这种情况可以通过开启逃逸分析参数(-XX:+DoEscapeAnalysis)来优化对象内存分配位置，JDK7之后默认开启逃逸分析，如果要关闭使用参数(-XX:-DoEscapeAnalysis)

#### 什么时候开始回收
安全点与安全区域
安全点就是指代码中一些特定的位置,当线程运行到这些位置时它的状态是确定的,这样JVM就可以安全的进行一些操作,比如GC等，所以GC不是想什么时候做就立即触发的，是需要等待所有线程运行到安全点后才能触发。 这些特定的安全点位置主要有以下几种: 
1. 方法返回之前；
2. 调用某个方法之后；
3. 抛出异常的位置；
4. 循环的末尾；

安全区域又是什么？ 
Safe Point 是对正在执行的线程设定的。如果一个线程处于Sleep 或中断状态，它就不能响应 JVM 的中断请求，再运行到 Safe Point上。 因此JVM引入了Safe Region。 Safe Region是指在一段代码片段中，引用关系不会发生变化。在这个区域内的任意地方开始GC都是安全的。
线程在进入Safe Region 的时候先标记自己已进入了 Safe Region，等到被唤醒时准备离开 Safe Region 时，先检查能否离开，如果GC完成了，那么线程可以离开，否则它必须等待直到收到安全离开的信号为止。

#### 如何回收
###### 回收算法的大致思想
标记-清除：分为标记、清除两个阶段，首先标记出所有需要回收的对象，标记完成后统一回收所有被标记的对象，它是最基础的收集算法
缺点：效率不高、会出现碎片

复制算法：将内存分为大小相同的两块，每次使用其中一块，当这一块的内存使用完后，就将还存活的对象复制到另一块，然后把使用空间一次清理掉。这样每次的内存回收都是对内存区的一半进行回收。

标记-整理：标记完后，将所有存活的对象向一端移动，然后直接清理掉端边界以外的内存。

分代收集算法：将堆分成年轻代，老年代，根据其特点使用不同的清理算法。

###### 垃圾收集器
是回收算法的具体实现。没有万能的垃圾收集器，我们能做的就是根据具体应用场景选择适合自己的垃圾收集器。但目前情况，常用的搭配是ParNew + CMS。

- Serial、Serial Old收集器（前一个是新生代采用复制算法，后一个是老年代采用标记-整理算法）`-XX:+UseSerialGC -XX:+UseSerialOldGC` 

Serial（串行）收集器是最基本、历史最悠久的垃圾收集器。它的 “单线程” 的意义不仅仅意味着它只会使用一条垃圾收集线程去完成垃 圾收集工作，更重要的是它在进行垃圾收集工作的时候必须暂停其他所有的工作线程（ "Stop The World" ），直到它收集结束。另外，Serial Old在以前的版本中与Parallel Scavenge收集器搭配使用，另一种用途是作为CMS收集器的后备方案。

- ParNew（Serial的多线程版本，新生代使用）`-XX:+UseParNewGC`

除了Serial收集器外，目前只有它能与CMS收集器配合工作。
自JDK 9开始，ParNew和CMS从此只能互相搭配使用，再也没有其他收集器能够和它们配合了。

- Parallel Scavenge收集器（新生代采用复制算法，老年代采用标记-整理算法）`-XX:+UseParallelGC - XX:+UseParallelOldGC`

经常被称作“吞吐量优先收集器”。提供了两个参数用于精确控制吞吐量，分别是控制最大垃圾收集停顿时间的`-XX：MaxGCPauseMillis`参数以及直接设置吞吐量大小的`-XX：GCTimeRatio`参数。注意，垃圾收集停顿时间缩短是以牺牲吞吐量和新生代空间为代价换取。
有一个参数-XX：+UseAdaptiveSizePolicy值得关注，可以自动调节新生代大小，Eden与Suvivor的比例等，这也称为垃圾收集的自适应的调节策略。但目前这个垃圾收集器在生产用的不多。

- CMS（老年代）`-XX:+UseConcMarkSweepGC` [实例：亿级流量电商实例](#jump1)

是一种以获取最短回收停顿时间为目标的收集器。它非常符合在注重用户体验的应用上使用，它是HotSpot虚拟机第一款真正意义上的并发收集器， 它第一次实现了让垃圾收集线程与用户线程（基本上）同时工作。
四个步骤： 
初始标记：暂停所有的其他线程，并记录下gc roots直接能引用的对象，速度很快；
并发标记：同时开启GC和用户线程，用一个闭包结构去记录可达对象。但在这个阶段结束，这个闭包结构并不能保证包含当前所有的可达对象。因为用户线程可能会不断的更新引用域，所以GC线程无法保证可达性分析的实时性。所以这个算法里会跟踪记录这些发生引用更新的地方；
重新标记：重新标记阶段就是为了修正并发标记期间因为用户程序继续运行而导致标记 产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段的时间稍长，远远比并发标记阶段时间短；
并发清理：开启用户线程，同时GC线程开始对未标记的区域做清扫。

问题：
1. CMS收集器对处理器资源非常敏感：在并发阶段，它虽然不会导致用户线程停顿，但却会因为占用了一部分线程（或者说处理器的计算能力）而导致应用程序变慢，降低总吞吐量（默认启动的回收线程数是（处理器核心数量+3）/4）；
2. 浮动垃圾：并发标记和并发清理阶段，用户线程是还在继续运行的，程序在运行自然就还会伴随有新的垃圾对象不断产生；
3. 会有大量空间碎片产生：这是一个标记清理的垃圾收集器。`-XX：+UseCMSCompactAtFullCollection`整理；
4. 会存在上一次垃圾回收还没执行完，然后垃圾回收又被触发的情况，特别是在并发标记和并发清理阶段会出现，一边回收，系统一边运行，也许没回收完就再次触发full gc，也就是"concurrent mode failure"，此时会进入stop the world，用serial old垃圾收集器来回收；

- G1（年轻代+老年代）`-XX:+UseG1GC` [实例：每秒几十万的并发](#jump2)

主要针对配备多颗处理器及大容量内存的机器。 以极高概率满足GC停顿时间要求的同时，还具备高吞吐量性能特征。
大致分为以下几个步骤： 
初始标记（initial mark，STW）：暂停所有的其他线程，并记录下gc roots直接能引用的对象，速度很快； 
并发标记（Concurrent Marking）：同CMS的并发标记；
最终标记（Remark，STW）：同CMS的重新标记；
筛选回收（Cleanup，STW）：筛选回收阶段首先对各个Region的回收价值和成本进行 排序，根据用户所期望的GC停顿时间(可以用JVM参数 `-XX:MaxGCPauseMillis`指定)来制定回收计划，比如说老年代此时有1000个Region都满了，但是因为根据预期停顿时间，本次垃圾回收可能只能停顿200毫秒，那么通过之前回收成本计算得知，可能回收其中800个Region刚好需要200ms，那么就只会回收800个Region，尽量把GC导致的停顿时间控制在我们指定的范围内。这个阶段其实也可以做到与用户程序一起并发执行，但是因为只回收一部分Region，时间是用户可控制的，而且停顿用户线程将大幅提高收集效率。回收算法主要用的是复制算法，将一个region中的存活对象复制到另一个region中，几乎不会有太多内存碎片。

YoungGC
YoungGC并不是说现有的Eden区放满了就会马上触发，而且G1会计算下现在Eden区回收大 概要多久时间，如果回收时间远远小于参数 -XX:MaxGCPauseMills 设定的值，那么增加年轻代的region，继续给新对象存放，不会马上做Young GC，直到下一次Eden区放满，G1计算回收时间接近参数 -XX:MaxGCPauseMills 设定的值，那么就会触发Young GC.
MixedGC
不是FullGC，老年代的堆占有率达到参数(-XX:InitiatingHeapOccupancyPercen)设定的值则触发，回收所有的Young和部分Old(根据期望的GC停顿时间确定old区垃圾收集的优先顺序)以及大对象区，正常情况G1的垃圾收集是先做MixedGC，主要使用复制算法，需要把各个region中存活的对象拷贝到别的region里去，拷贝过程中如果发现没有足够的空region能够承载拷贝对象就会触发一次Full GC 
Full GC
停止系统程序，然后采用单线程进行标记、清理和压缩整理，好空闲出来一批Region来供下一次MixedGC使用，这个过程是非常耗时的。

参数：
`-XX:+UseG1GC`：使用G1收集器
`-XX:ParallelGCThreads`：指定GC工作的线程数量 
`-XX:G1HeapRegionSize`：指定分区大小(1MB~32MB，且必须是2的幂)，默认将整堆划分为 2048个分区
`-XX:MaxGCPauseMillis`：目标暂停时间(默认200ms) 
`-XX:G1NewSizePercent`：新生代内存初始空间(默认整堆5%) 
`-XX:G1MaxNewSizePercent`：新生代内存最大空间 
`-XX:TargetSurvivorRatio`：Survivor区的填充容量(默认50%)，Survivor区域里的一批对象(年龄 1+年龄2+年龄n的多个年龄对象)总和超过了Survivor区域的50%，此时就会把年龄n(含)以上的对 象都放入老年代
`-XX:MaxTenuringThreshold`：最大年龄阈值(默认15) 
`-XX:InitiatingHeapOccupancyPercent`：老年代占用空间达到整堆内存阈值(默认45%)，则执行新生代和老年代的混合收集(MixedGC)，比如我们之前说的堆默认有2048个region，如果有接近 1000个region都是老年代的region，则可能就要触发MixedGC了 
`-XX:G1HeapWastePercent(默认5%)`：gc过程中空出来的region是否充足阈值，在混合回收的时候，对Region回收都是基于复制算法进行的，都是把要回收的Region里的存活对象放入其他Region，然后这个Region中的垃圾对象全部清理掉，这样的话在回收过程就会不断空出来新的Region，一旦空闲出来的Region数量达到了堆内存的5%，此时就会立即停止混合回收，意味着本次混合回收就结束了。
`-XX:G1MixedGCLiveThresholdPercent`：(默认85%) region中的存活对象低于这个值时才会回收该region，如果超过这个值，存活对象过多，回收的的意义不大。 
`-XX:G1MixedGCCountTarget`:在一次回收过程中指定做几次筛选回收(默认8次)，在最后一个筛选回收阶段可以回收一会，然后暂停回收，恢复系统运行，一会再开始回收，这样可以让系统不至于单次停顿时间过长。

什么场景适合使用G1
- 50%以上的堆被存活对象占用
- 对象分配和晋升的速度变化非常大
- 垃圾回收时间特别长，超过1秒
- 8G以上的堆内存
- 停顿时间是500ms以内

#### 垃圾回收的分配机制
Minor GC/Young GC：年轻代的回收
Full GC：年轻代+老年代的回收

###### 优先在Eden分配
对象在新生代中Eden区分配。当Eden区没有足够空间进行分配时，虚拟机将发起一次Minor GC。

###### 大对象直接进入老年代
为了避免为大对象分配内存时的复制操作而降低效率。

大对象就是需要大量连续内存空间的对象（比如：字符串、数组）。JVM参数 - XX:PretenureSizeThreshold 可以设置大对象的大小，如果对象超过设置大小会直接进入老年代，不会进入年轻代，这个参数只在Serial和ParNew两个收集器下有效。 比如设置JVM参数：-XX:PretenureSizeThreshold=1000000 - XX:+UseSerialGC。

###### 长期存活的对象将进入老年代
虚拟机采用了分代收集的思想来管理内存，虚拟机给每个对象一个对象年龄（Age）计数器（对象头中）。对象在Eden出生并经过第一次 Minor GC 后仍然能够存活，并且能被 Survivor 容纳的话，将被移动到 Survivor 空间中，并将对象年龄设为1。对象 在 Survivor 中每熬过一次 MinorGC，年龄就增加1岁，当它的年龄增加到一定 程度（默认为15岁），就会被晋升到老年代中。对象晋升到老年代的年龄阈 值，可以通过参数 -XX:MaxTenuringThreshold 来设置。

###### 对象动态年龄判断
为了能更好地适应不同程序的内存状况，HotSpot虚拟机并不是永远要求对象的年龄必须达到-XX：MaxTenuringThreshold才能晋升老年代，如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代。
实际上，如果Minor GC后（Eden + From区清理完放到To区），年龄1+年龄2+年龄n的多个年龄对象总和超过了Survivor区域的50%，此时就会把年龄n以上的对象都放入老年代。这个规则是希望那些可能是长期存活的对象，尽早进入老年代。

###### 老年代空间分配担保机制
年轻代每次minor gc之前JVM都会计算下老年代剩余可用空间 如果这个可用空间小于年轻代里现有的所有对象大小之和(包括垃圾对象) 就会看“-XX:-HandlePromotionFailure”(jdk1.8默认设置了)的参数是否设置了。
如果有这个参数，就会看看老年代的可用内存大小，是否大于之前每一次minor gc后进入老年代的对象的平均大小。如果上一步结果是小于或者之前说的参数没有设置，那么就会触发一次Full gc，对老年代和年轻代一起回收一次垃圾，如果回收完还是没有足够空间存放 新的对象就会发生"OOM"。
当然，如果minor gc之后剩余存活的需要挪动到老年代的对象大小还是大于老年代可用空间，那么也会触发full gc，full gc完之后如果还是没用空间放minor gc之后的存活对象，则也会发生“OOM”。

#### 实例
<span id="jump1">亿级流量电商：ParNem + CMS</span>
点击量是亿级的，如果按每个用户平均20次点击，那么算出日活用户是500W的量，付费转化率按10%计算，日均产生50W订单。
平时会在3~4个小时产生出来，每秒在几十单左右；但大促活动会在几分钟内产生，每秒1000单左右。
这个时候估算每秒产生的对象的大小，多久后会成为垃圾对象？

对于8G内存，参数设置为` ‐Xms3072M ‐Xmx3072M ‐Xmn1536M ‐Xss1M ‐XX:PermSize=256M ‐XX:MaxPermSize=256M ‐XX:SurvivorRatio=8`

系统按每秒生成60MB的速度来生成对象，大概运行20秒就会撑满eden区，会出发minor gc，大 概会有95%以上对象成为垃圾被回收，可能最后一两秒生成的对象还被引用着，我们暂估为 100MB左右，那么这100M会被挪到S0区，回忆下动态对象年龄判断原则，这100MB对象同龄而且总和是否大于S0区的50%，是否要调整Suvivor的大小。

<span id="jump2">每秒几十万的并发</span>
kafka每秒就处理几万甚至几十万的消息，一般来说都需要大内存的机器来部署（64G）也就是说可以给年轻代分配三四十G的内存用来支撑高并发处理。使用G1收集器，设置 -XX:MaxGCPauseMills为50ms，假设50ms能够回收三到四个G内存，然后50ms的卡顿其实完全能够接受，用户几乎无感知，那么整个系统就可以在卡顿几乎无感知的情况下一边处理业务一边收集垃圾。 G1天生就适合这种大内存机器的JVM运行，可以比较完美的解决大内存垃圾回收时间过长的问题。

#### 如何选择垃圾收集器 
1. 优先调整堆的大小让服务器自己来选择；
2. 如果内存小于100M，使用串行收集器；
3. 如果是单核，并且没有停顿时间的要求，串行或JVM自己选择；
4. 如果允许停顿时间超过1秒，选择并行或者JVM自己选；
5. 如果响应时间最重要，并且不能超过1秒，使用并发收集器；

## JVM调优
> 一般方法
#### jps
查看进程

#### jmap
`jmap -histo pid > /home/log.txt`
num：序号
instances：实例数量
bytes：占用空间大小 
class name：类名称，[C is a char[]，[S is a short[]，[I is a int[]，[B is a byte[]，[[I is a int[][]

查看堆：`jmap -heap pid`

堆内存dump操作
`jmap -dump:format=b,file=eureka.hporf pid`
也可以设置内存溢出自动导出dump文件(内存很大的时候，可能会导不出来) `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./`dunmp出来的可以用jvisualvm打开分析

#### jstack
可以查找死锁：`jstak pid`

找出占用cpu最高的堆栈信息 
1. 使用命令top -p < pid> ，显示你的java进程的内存情况，pid是你的java进程号，比如4977；
2. 按H，获取每个线程的内存情况；
3. 找到内存和cpu占用最高的线程tid，比如4977；
4. 转为十六进制得到 0x1371 ,此为线程id的十六进制表示；
5. 执行 jstack 4977|grep -A 10 1371，得到线程堆栈信息中1371这个线程所在行的后面10行；
6. 查看对应的堆栈信息找出可能存在问题的代码；

#### 远程连接jvisualvm
影响性能，线上都是关闭的，实验看看就好
启动普通的jar程序JMX端口配置： `java -Dcom.sun.management.jmxremote.port=8899 -Dcom.sun.management.jmxremote.ssl=false - Dcom.sun.management.jmxremote.authenticate=false -jar foo.jar`
tomcat的JMX配置：`JAVA_OPTS=-Dcom.sun.management.jmxremote.port=8899 -Dcom.sun.management.jmxremote.ssl=false - Dcom.sun.management.jmxremote.authenticate=false`
jvisualvm远程连接服务需要在远程服务器上配置host(连接ip 主机名)，并且要关闭防火墙

#### jinfo
查看正在运行的Java应用程序的扩展参数：`jinfo -flags pid`

#### jstat
jstat命令可以查看堆内存各部分的使用量，以及加载类的数量。命令的格式如下： jstat [-命令选项] [vmid] [间隔时间(毫秒)] [查询次数]

###### 垃圾回收统计
`jstat -gc pid` 最常用，可以评估程序内存使用及GC压力整体情况
S0C：第一个幸存区的大小；
S1C：第二个幸存区的大小；
S0U：第一个幸存区的使用大小；
S1U：第二个幸存区的使用大小；
EC：伊甸园区的大小；
EU：伊甸园区的使用大小；
OC：老年代大小；
OU：老年代使用大小；
MC：方法区大小(元空间)；
MU：方法区使用大小；
CCSC:压缩类空间大小；
CCSU:压缩类空间使用大小；
YGC：年轻代垃圾回收次数；
YGCT：年轻代垃圾回收消耗时间，单位s；
FGC：老年代垃圾回收次数；
FGCT：老年代垃圾回收消耗时间，单位s；
GCT：垃圾回收消耗总时间，单位s；

###### 堆内存统计
`jstat -gccapacity pid`
NGCMN：新生代最小容量；
NGCMX：新生代最大容量；
NGC：当前新生代容量；
S0C：第一个幸存区大小；
S1C：第二个幸存区的大小；
EC：伊甸园区的大小；
OGCMN：老年代最小容量；
OGCMX：老年代最大容量；
OGC：当前老年代大小；
OC:当前老年代大小；
MCMN:最小元数据容量；
MCMX：最大元数据容量；
MC：当前元数据空间大小；
CCSMN：最小压缩类空间大小；
CCSMX：最大压缩类空间大小；
CCSC：当前压缩类空间大小；
YGC：年轻代gc次数；
FGC：老年代GC次数；

###### 新生代垃圾回收统计
`jstat -gcnew pid`
S0C：第一个幸存区的大小；
S1C：第二个幸存区的大小；
S0U：第一个幸存区的使用大小；
S1U：第二个幸存区的使用大小；
TT:对象在新生代存活的次数；
MTT:对象在新生代存活的最大次数；
DSS:期望的幸存区大小；
EC：伊甸园区的大小；
EU：伊甸园区的使用大小；
YGC：年轻代垃圾回收次数；
YGCT：年轻代垃圾回收消耗时间 

###### 新生代内存统计
`jstat -gcnewcapacity pid`
NGCMN：新生代最小容量；
NGCMX：新生代最大容量；
NGC：当前新生代容量；
S0CMX：最大幸存1区大小；
S0C：当前幸存1区大小；
S1CMX：最大幸存2区大小；
S1C：当前幸存2区大小；
ECMX：最大伊甸园区大小；
EC：当前伊甸园区大小；
YGC：年轻代垃圾回收次数；
FGC：老年代回收次数；

###### 老年代垃圾回收统计
`jstat -gcold pid`
MC：方法区大小；
MU：方法区使用大小；
CCSC:压缩类空间大小；
CCSU:压缩类空间使用大小；
OC：老年代大小；
OU：老年代使用大小；
YGC：年轻代垃圾回收次数；
FGC：老年代垃圾回收次数；
FGCT：老年代垃圾回收消耗时间；
GCT：垃圾回收消耗总时间；

###### 老年代内存统计
`jstat -gccapacity pid`
OGCMN：老年代最小容量；
OGCMX：老年代最大容量；
OGC：当前老年代大小；
OC：老年代大小；
YGC：年轻代垃圾回收次数；
FGC：老年代垃圾回收次数；
FGCT：老年代垃圾回收消耗时间；
GCT：垃圾回收消耗总时间；

###### 元数据空间统计
`jstat -gcmetacapacity pid`
MCMN:最小元数据容量；
MCMX：最大元数据容量；
MC：当前元数据空间大小；
CCSMN：最小压缩类空间大小；
CCSMX：最大压缩类空间大小；
CCSC：当前压缩类空间大小；
YGC：年轻代垃圾回收次数；
FGC：老年代垃圾回收次数；
FGCT：老年代垃圾回收消耗时间；
GCT：垃圾回收消耗总时间；

###### ？
`jstat -gcutil pid`
S0：幸存1区当前使用比例；
S1：幸存2区当前使用比例；
E：伊甸园区使用比例；
O：老年代使用比例；
M：元数据区使用比例；
CCS：压缩使用比例；
YGC：年轻代垃圾回收次数；
FGC：老年代垃圾回收次数；
FGCT：老年代垃圾回收消耗时间；
GCT：垃圾回收消耗总时间；

#### JVM运行情况预估 
用jstat gc -pid 命令可以计算出如下一些关键数据，有了这些数据就可以采用之前介绍过的优化思路，先给自己的系统设置一些初始性的JVM参数，比如堆内存大小，年轻代大小，Eden和Survivor的比例，老年代的大小，大对象的阈值，大龄对象进入老年代的阈值等。
 
###### 年轻代对象增长的速率
可以执行命令 jstat -gc pid 1000 10 (每隔1秒执行1次命令，共执行10次)，通过观察EU(eden区的使用)来估算每秒eden大概新增多少对象，如果系统负载不高，可以把频率1秒换成1分钟，甚至10分钟来观察整体情况。注意，一般系统可能有高峰期和日常期，所以需要在不 同的时间分别估算不同情况下对象增长速率。

###### Young GC的触发频率和每次耗时
知道年轻代对象增长速率我们就能推根据eden区的大小推算出Young GC大概多久触发一次，Young GC的平均耗时可以通过 YGCT/YGC公式算出，根据结果我们大概就能知道系统大概多久会因为Young GC的执行而卡顿多久。 
 
###### 每次Young GC后有多少对象存活和进入老年代
这个因为之前已经大概知道Young GC的频率，假设是每5分钟一次，那么可以执行命令 jstat -gc pid 300000 10 ，观察每次结果eden，survivor和老年代使用的变化情况，在每次gc后eden区使用一般会大幅减少，survivor和老年代都有可能增长，这些增长的对象就是每次 Young GC后存活的对象，同时还可以看出每次Young GC后进去老年代大概多少对象，从而可以推算出老年代对象增长速率。

###### Full GC的触发频率和每次耗时
知道了老年代对象的增长速率就可以推算出Full GC的触发频率了，Full GC的每次耗时可以用公式 FGCT/FGC 计算得出。 

其实简单来说优化思路就是尽量让每次Young GC后的存活对象小于Survivor区域的50%，都留存在年轻代里。尽量别让对象进入老年代。尽量减少Full GC的频率，避免频繁Full GC对JVM性能的影响。

###### 实例
系统频繁Full GC导致系统卡顿是怎么回事
机器配置：2核4G 
JVM内存大小：2G
系统运行时间：7天 
期间发生的Full GC次数和耗时：500多次，200多秒
期间发生的Young GC次数和耗时：1万多次，500多秒
大致算下来每天会发生70多次Full GC，平均每小时3次，每次Full GC在400毫秒左右； 
每天会发生1000多次Young GC，每分钟会发生1次，每次Young GC在50毫秒左右。


#### GC日志
对于java应用我们可以通过一些配置把程序运行过程中的gc日志全部打印出来，然后分析gc日志得到关键性指标，分析GC原因，调优JVM参数。
打印GC日志方法，在JVM参数里增加参数 `‐XX:+PrintGCDetails ‐XX:+PrintGCTimeStamps ‐XX:+PrintGCDateStamps ‐Xloggc:./gc.log`

[工具分析GC日志](https://gceasy.io/)
一般根据业务情况分析调优

## Class文件
class常量池，是Class文件中的资源仓库。 Class文件中除了包含类的版本、字段、方法、接口等描述信息外， 还有一项信息就是常量池(constant pool table)，用于存放编译器生成的各种字面量(Literal)和符号引用(Symbolic References)。

一个class文件16进制大体结构是很复杂的：
| 魔数 | 次版本 | 主版本 | 常量池计数器 | 常量池数据 |
| - | - | - | - | - |
|cafe babe| 0000 | 0034（十进制 52，JDK 1.8） | 0042 | ... |

一般可以通过javap命令生成更可读的JVM字节码指令文件，-v是更详细的输出：`javap -v a.class`
```
Classfile /Users/apple/test/common/PinYinUtil.class
  Last modified 2020-4-8; size 1313 bytes
  MD5 checksum c9b13a288196f0fc9c2057e2cb393f69
  Compiled from "PinYinUtil.java"
public class com.common.PinYinUtil
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #12.#36        // java/lang/Object."<init>":()V
   #2 = Methodref          #37.#38        // java/lang/String.trim:()Ljava/lang/String;
   #3 = Methodref          #37.#39        // java/lang/String.charAt:(I)C
   #4 = Methodref          #40.#41        // java/lang/Character.toUpperCase:(C)C
   #5 = Methodref          #40.#42        // java/lang/Character.valueOf:(C)Ljava/lang/Character;
   #6 = Methodref          #43.#44        // net/sourceforge/pinyin4j/PinyinHelper.toHanyuPinyinStringArray:(C)[Ljava/lang/String;
   #7 = Class              #45            // java/lang/IllegalArgumentException
   #8 = Methodref          #7.#46         // java/lang/IllegalArgumentException."<init>":(Ljava/lang/String;)V
   #9 = Class              #47            // com/common/PinYinUtil
  #10 = Methodref          #48.#49        // org/slf4j/LoggerFactory.getLogger:(Ljava/lang/Class;)Lorg/slf4j/Logger;
  #11 = Fieldref           #9.#50         // com/common/PinYinUtil.log:Lorg/slf4j/Logger;
  #12 = Class              #51            // java/lang/Object
  #13 = Utf8               log
  #14 = Utf8               Lorg/slf4j/Logger;
  #15 = Utf8               <init>
  #16 = Utf8               ()V
  #17 = Utf8               Code
  #18 = Utf8               LineNumberTable
  #19 = Utf8               LocalVariableTable
  #20 = Utf8               this
  #21 = Utf8               Lcom/common/PinYinUtil;
  #22 = Utf8               firstLetter
  #23 = Utf8               (Ljava/lang/String;)Ljava/lang/Character;
  #24 = Utf8               pinyinArray
  #25 = Utf8               [Ljava/lang/String;
  #26 = Utf8               result
  #27 = Utf8               C
  #28 = Utf8               word
  #29 = Utf8               Ljava/lang/String;
  #30 = Utf8               StackMapTable
  #31 = Class              #25            // "[Ljava/lang/String;"
  #32 = Utf8               MethodParameters
  #33 = Utf8               <clinit>
  #34 = Utf8               SourceFile
  #35 = Utf8               PinYinUtil.java
  #36 = NameAndType        #15:#16        // "<init>":()V
  #37 = Class              #52            // java/lang/String
  #38 = NameAndType        #53:#54        // trim:()Ljava/lang/String;
  #39 = NameAndType        #55:#56        // charAt:(I)C
  #40 = Class              #57            // java/lang/Character
  #41 = NameAndType        #58:#59        // toUpperCase:(C)C
  #42 = NameAndType        #60:#61        // valueOf:(C)Ljava/lang/Character;
  #43 = Class              #62            // net/sourceforge/pinyin4j/PinyinHelper
  #44 = NameAndType        #63:#64        // toHanyuPinyinStringArray:(C)[Ljava/lang/String;
  #45 = Utf8               java/lang/IllegalArgumentException
  #46 = NameAndType        #15:#65        // "<init>":(Ljava/lang/String;)V
  #47 = Utf8               com/common/PinYinUtil
  #48 = Class              #66            // org/slf4j/LoggerFactory
  #49 = NameAndType        #67:#68        // getLogger:(Ljava/lang/Class;)Lorg/slf4j/Logger;
  #50 = NameAndType        #13:#14        // log:Lorg/slf4j/Logger;
  #51 = Utf8               java/lang/Object
  #52 = Utf8               java/lang/String
  #53 = Utf8               trim
  #54 = Utf8               ()Ljava/lang/String;
  #55 = Utf8               charAt
  #56 = Utf8               (I)C
  #57 = Utf8               java/lang/Character
  #58 = Utf8               toUpperCase
  #59 = Utf8               (C)C
  #60 = Utf8               valueOf
  #61 = Utf8               (C)Ljava/lang/Character;
  #62 = Utf8               net/sourceforge/pinyin4j/PinyinHelper
  #63 = Utf8               toHanyuPinyinStringArray
  #64 = Utf8               (C)[Ljava/lang/String;
  #65 = Utf8               (Ljava/lang/String;)V
  #66 = Utf8               org/slf4j/LoggerFactory
  #67 = Utf8               getLogger
  #68 = Utf8               (Ljava/lang/Class;)Lorg/slf4j/Logger;
{
  public com.common.PinYinUtil();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 10: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/common/PinYinUtil;

  public static java.lang.Character firstLetter(java.lang.String);
    descriptor: (Ljava/lang/String;)Ljava/lang/Character;
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=4, args_size=1
         0: aload_0
         1: invokevirtual #2                  // Method java/lang/String.trim:()Ljava/lang/String;
         4: iconst_0
         5: invokevirtual #3                  // Method java/lang/String.charAt:(I)C
         8: istore_1
         9: iload_1
        10: bipush        97
        12: if_icmplt     29
        15: iload_1
        16: bipush        122
        18: if_icmpgt     29
        21: iload_1
        22: invokestatic  #4                  // Method java/lang/Character.toUpperCase:(C)C
        25: invokestatic  #5                  // Method java/lang/Character.valueOf:(C)Ljava/lang/Character;
        28: areturn
        29: iload_1
        30: bipush        65
        32: if_icmplt     46
        35: iload_1
        36: bipush        90
        38: if_icmpgt     46
        41: iload_1
        42: invokestatic  #5                  // Method java/lang/Character.valueOf:(C)Ljava/lang/Character;
        45: areturn
        46: iload_1
        47: bipush        48
        49: if_icmplt     63
        52: iload_1
        53: bipush        57
        55: if_icmpgt     63
        58: iload_1
        59: invokestatic  #5                  // Method java/lang/Character.valueOf:(C)Ljava/lang/Character;
        62: areturn
        63: iload_1
        64: invokestatic  #6                  // Method net/sourceforge/pinyin4j/PinyinHelper.toHanyuPinyinStringArray:(C)[Ljava/lang/String;
        67: astore_2
        68: aload_2
        69: ifnull        78
        72: aload_2
        73: arraylength
        74: iconst_1
        75: if_icmpge     87
        78: new           #7                  // class java/lang/IllegalArgumentException
        81: dup
        82: aload_0
        83: invokespecial #8                  // Method java/lang/IllegalArgumentException."<init>":(Ljava/lang/String;)V
        86: athrow
        87: aload_2
        88: iconst_0
        89: aaload
        90: iconst_0
        91: invokevirtual #3                  // Method java/lang/String.charAt:(I)C
        94: istore_3
        95: iload_3
        96: invokestatic  #4                  // Method java/lang/Character.toUpperCase:(C)C
        99: invokestatic  #5                  // Method java/lang/Character.valueOf:(C)Ljava/lang/Character;
       102: areturn
      LineNumberTable:
        line 12: 0
        line 13: 9
        line 14: 21
        line 15: 29
        line 16: 41
        line 17: 46
        line 18: 58
        line 20: 63
        line 21: 68
        line 22: 78
        line 24: 87
        line 25: 95
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
           68      35     2 pinyinArray   [Ljava/lang/String;
           95       8     3 result   C
            0     103     0  word   Ljava/lang/String;
            9      94     1 firstLetter   C
      StackMapTable: number_of_entries = 5
        frame_type = 252 /* append */
          offset_delta = 29
          locals = [ int ]
        frame_type = 16 /* same */
        frame_type = 16 /* same */
        frame_type = 252 /* append */
          offset_delta = 14
          locals = [ class "[Ljava/lang/String;" ]
        frame_type = 8 /* same */
    MethodParameters:
      Name                           Flags
      word

  static {};
    descriptor: ()V
    flags: ACC_STATIC
    Code:
      stack=1, locals=0, args_size=0
         0: ldc           #9                  // class com/common/PinYinUtil
         2: invokestatic  #10                 // Method org/slf4j/LoggerFactory.getLogger:(Ljava/lang/Class;)Lorg/slf4j/Logger;
         5: putstatic     #11                 // Field log:Lorg/slf4j/Logger;
         8: return
      LineNumberTable:
        line 9: 0
}
```

Constant pool是常量池（静态常量池），主要存放两大类常量：字面量和符号引用。
只有到运行时被加载到内存后，这些符号才有对应的内存地址信息，这些常量池一旦被装入内存就变成运行时常量池了，对应的符号引用在程序加载或运行时会被转变为被加载到内存区域的代码的直接引用，也就是我们说的动态链接了。

#### 字面量
字面量就是指由字母、数字等构成的字符串或者数值常量，字面量只可以右值出现，所谓右值是指等号右边的值，如：int a=1 这里的a为左值，1为右值

#### 符号引用
符号引用是编译原理中的概念，是相对于直接引用来说的。主要包括了以下三类常量： 类和接口的全限定名、字段的名称和描述符、方法的名称和描述符


#### 字符串常量池
字符串常量池的设计思想
1. 字符串的分配，和其他的对象分配一样，耗费高昂的时间与空间代价，作为最基础的数据类型，大量频繁的创建字符串，极大程度地影响程序的性能；
2. JVM为了提高性能和减少内存开销，在实例化字符串常量的时候进行了一些优化 
    - 为字符串开辟一个字符串常量池，类似于缓存区；
    - 创建字符串常量时，首先检查字符串常量池是否存在该字符串；
    - 存在该字符串，返回引用实例，不存在，实例化该字符串并放入池中；

`String str4 = new String(“abc”)`创建多少个对象？ 
1. 在常量池中查找是否有“abc”对象 有则返回对应的引用实例 没有则在常量池中创建对应的实例对象;
2. 在堆中 new 一个 String("abc") 对象;
3. 将对象地址赋值给str4，创建一个引用;

java中基本类型的包装类的大部分都实现了常量池技术，这些类是Byte，Short，Integer，Long，Character，Boolean，两种浮点数类型的包装类则没有实现。另外Byte，Short，Integer，Long，Character这5种整型的包装类也只是在对应值小于等于127时才可使用对象池，也即对象不负责创建和管理大于127的这些类的对象。

String.intern()：返回一个常量池里面的字符串，就是一个在字符串常量池中有了一个入口。如果 以前没有在字符串常量池中，那么它就会被添加到里面。