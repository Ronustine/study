# 是什么？

项目管理，用于构建项目，管理，Jar包下载

## 怎么用？

与本机的JDK版本有关，所以事先应该配置好JDK的环境变量
apache官网下载Maven -> 解压 -> 将bin目录写入环境变量 -> cmd名录输入mvn -v -> 有版本说明安装成功
在cmd执行mvn help:system，这时下载依赖的jar包，会在C:\Users\Administrator下生成一个.m的目录，将conf下的setting.xml放入

## Maven标准目录结构

| 路径 | 说明 |
|--|--|
| src/main/java | Application/Library sources|
| src/main/resources | Application/Library resources|
| src/main/filters | Resource filter files |
| src/main/assembly | Assembly descriptors |
| src/main/config | Configuration files |
| src/main/scripts | Application/Library scripts |
| src/main/webapp | Web application sources |
| src/test/java | Test sources |
| src/test/resources | Test resources |
| src/test/filters | Test resource filter files |
| src/site | Site |

如何搭建多环境

```xml
<profiles>
    <profile>
        <id>dev</id>
        <properties>
            <profiles.active>dev</profiles.active>
            <maven.resources.overwrite>true</maven.resources.overwrite>
        </properties>
    </profile>
    <profile>
        <!-- 生产环境 -->
        <id>prod</id>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
        <properties>
            <profiles.active>prod</profiles.active>
            <maven.resources.overwrite>true</maven.resources.overwrite>
        </properties>
    </profile>
</profiles>
```