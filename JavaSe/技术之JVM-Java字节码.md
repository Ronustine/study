### 目录

- [字节码](#1)
- [组成](#2)
- [Java内存模型](#3)

### [字节码](#1)

将Java编译成class文件

### [组成](#2)

> Magical Number, Version, Constant Pool, Access Flags, This Class Name, Super Class Name, Interface, Fields, Methods, Attributes
每个大小均用字节表示，u1(一个字节)，u2，u4，u8

#### 魔数

u4，魔数是用来区分文件类型的一种标志，一般都是用文件的前几个字节来表示。比如0XCAFE BABE表示的是class文件，文件类型虽然可以通过文件名后缀来判断，但文件名是可以修改的（包括后缀），那么为了保证文件的安全性，讲文件类型写在文件内部来保证不被篡改。
从java的字节码文件类型我们看到，魔数是CAFE BABE。

#### 版本号

u2 + u2，分成两个，minor_version（次版本）和major_version（主版本）。主版本51对应的正式jdk1.7。

#### 常量池

u2 + ??，第一个是常量池大小，第二个是常量池内容。
常量池是Class文件中的资源仓库，在接下来的内容中我们会发现很多地方会涉及，如Class Name，Interfaces等。常量池中主要存储2大类常量：字面量和符号引用。字面量如文本字符串，java中声明为final的常量值等等，而符号引用如类和接口的全局限定名，字段的名称和描述符，方法的名称和描述符。
常量池项目表：
// TODO 

描述了11中数据类型的结构，其实在jdk1.7之后又增加了3种(CONSTANT_MethodHandle_info,CONSTANT_MethodType_info 以及 CONSTANT_InvokeDynamic_info)，这样算起来一共是14种。
**注意：** 常量池索引从1开始，不是0，0作为空索引的保留

#### Access_Flag 访问标志

访问标志信息包括该Class文件是类还是接口，是否被定义成public，是否是abstract，如果是类，是否被声明成final。通过上面的源代码，我们知道该文件是类并且是public。

#### 类索引

类索引用于确定类的全限定名

#### 父类索引

除了Object，对象都会有一个父类索引

#### 接口索引

这个接口有2+n个字节，前两个字节表示的是接口数量，后面跟着就是接口的表

#### 字段表集合

字段表用于描述类和接口中声明的变量。这里的字段包含了类级别变量以及实例变量，但是不包括方法内部声明的局部变量。
结构：
| 类型 | 名称 | 数量 |
|---|---|---|
| u2 | access_flags | 1 |
| u2 | name_index | 1 |
| u2 | descriptor_index | 1 |
| u2 | attribute_count | 1 |
| attribute_info | attributes | attribute_count |

#### 方法

结构：
| 类型 | 名称 | 数量 |
|---|---|---|
| u2 | access_flags | 1 |
| u2 | name_index | 1 |
| u2 | descriptor_index | 1 |
| u2 | attribute_count | 1 |
| attribute_info | attributes | attribute_count |

#### 属性

结构：
| 类型 | 名称 | 数量 |
|---|---|---|
| u2 | attribute_name_index | 1 |
| u4 | attribute_length | 1 |
| u2 | sourcefile_index | 1 |
