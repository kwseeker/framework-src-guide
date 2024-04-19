# Mini-jvm

Github Repo: [guxingke/mini-jvm](https://github.com/guxingke/mini-jvm)

Mini-jvm是参考JDK8 HotSpot使用Java实现的，源码仅1W多行，相对于读HotSpot源码容易很多。

通过这个只是简单学下JVM设计思想和工作流程，后面有时间有精力再深入研究HotSpot源码。



## 启动调试

想了解应该怎么启动调试可以参考Args类对命令行参数的解析。

Mini-jvm 入口是 jvm-core 模块的 Main.java。

支持通过 -cp -jar 执行 Java 程序。

源码 example 模块中有一些示例代码，这里用这些代码调试；

### -cp 方式调试

1. 先准备`应用程序`和`rt.jar`

   比如先编译 Hello.java 获取 Hello.class；

   然后打包mini-jdk模块，生成 rt.jar。

2. 配置 IDEA 启动器：

   `-cp`： jvm-core
   `主类`： com.gxk.jvm.Main
   `程序参数`：-verbose -cp example/target/classes Hello

   调试的话不需要通过 -jar 引入Mini-jvm rt.jar 包，Mini-jvm会自动搜索 mini-jvm/mini-jdk/target/rt.jar，只需要提前打包好。



