
1. 将jdk调至1.6；
2. 导出，以Runable JAR file的形式打成jar包（需要写main方法才能导出）；
3. 使用工具ikmvc1.7，命令行运行ikvmc -target:library XXX.jar；
4. 调用dll时需要引入ikmvc相关的dll才能使用，如IKVM.OpenJDK.Corba.dll，IKVM.Runtime.dll，IKVM.Runtime.JNI.dll

转dll时出现的问题：
1. 如果出现warning IKVMC0108的错误则需要注意，dll是不能使用的，上面步骤应该不会出现；
2. 如果是一些Junit、测试的方法找不到类，可以忽视；
3. 高版本jdk生成的jar包转dll会失败（报错：class format error "52.0"）；