# 多网络接口环境Dubbo服务注册IP网段不一致问题与解决

如果服务器装了多网卡、或者配置了多Docker网络，有可能在系统初始化阶段注册Dubbo RPC服务时，给RPC服务注册上不同网段的IP地址，导致通信失败。

最好是在配置中指明统一使用的网络接口。

**解决方法**（解析源码发现可以通过下面方法解决）：

+ **方法1**

  通过`dubbo.protocol.host`直接传一个有效的IP地址；有效的IP地址：”空字符串、localhost、0.0.0.0、127.\*.\*.*” 除外；

+ **方法2** （**推荐**，可以通过保持各个环境网络网络接口名一致，避免修改配置代码）

  通过“dubbo.network.interface.preferred”指定优先使用的网络接口（NetworkInterface）名称；然后通过NetUtils自动获取此网络接口IP地址。
  
  ```properties
  -Ddubbo.network.interface.preferred=docker0
  ```

**下面分析IP地址来源原理**，参考 dubbo3-workflow.drawio。

调用栈：

```ini
#这一步会发现直接返回了NetUtils类静态变量LOCAL_ADDRESS，因为日志有调用此工具类提前解析过本地IP地址
getLocalHost:256, NetUtils (org.apache.dubbo.common.utils)
findConfiguredHosts:1027, ServiceConfig (org.apache.dubbo.config)
...
doExportUrlsFor1Protocol:594, ServiceConfig (org.apache.dubbo.config)
...
exportServices:424, DefaultModuleDeployer (org.apache.dubbo.config.deploy)
startSync:174, DefaultModuleDeployer (org.apache.dubbo.config.deploy)
start:156, DefaultModuleDeployer (org.apache.dubbo.config.deploy)
```

NetUtils之前读取IP地址原理：

1. 依次从系统环境变量 **TRI_DUBBO_IP_TO_BIND**、**DUBBO_IP_TO_BIND** 取RPC host 值；
2. 为空就从 **ProtocolConfig** （“dubbo.protocol.host”）配置对象中取 host,还是为空就从 **ProviderConfig** （”dubbo.provider.host“）配置对象中取 host；
3. 最后还要对host进行**有效性检查**，null、空字符串、`localhost`、`0.0.0.0`、`127.*.*.*` 都是无效的host；
4. 如果没有获取到有效的IP地址，需要再通过NetUtils获取本地网络接口IP。

NetUtils解析本地IP地址原理：

```java
//1 借助JDK网络API NetworkInterface 读取本地所有网络接口
Enumeration<NetworkInterface> interfaces = NetworkInterface.getNetworkInterfaces();
//2 过滤掉 Loopback、虚拟网络、未启用的网络接口
networkInterface.isLoopback()
    || networkInterface.isVirtual()
    || !networkInterface.isUp()
//3 过滤掉系统环境变量指定要忽略的网络接口（值是网络接口名称，也可以是正则表达式，多个名称以英文逗号分隔）
String DUBBO_NETWORK_IGNORED_INTERFACE = "dubbo.network.interface.ignored";
//	matched为true,就忽略此网络接口
matched = networkInterfaceDisplayName.matches(trimIgnoredInterface);
//  与要忽略的网络接口名相同也忽略
networkInterfaceDisplayName.equals(trimIgnoredInterface)
//4 优先使用 dubbo.network.interface.preferred 系统环境变量指定的网络接口
String DUBBO_PREFERRED_NETWORK_INTERFACE = "dubbo.network.interface.preferred";
Objects.equals(networkInterface.getDisplayName(), preferredNetworkInterface);
//5 最后尝试获取网络接口IP地址，如果有设置”java.net.preferIPv6Addresses“，就使用IPv6地址，否则使用IPv4地址，
// 	过滤掉无法获取IP地址的网络接口，返回第一个可用的地址
Enumeration<InetAddress> addresses = networkInterface.getInetAddresses();
if (address instanceof Inet6Address) {
    Inet6Address v6Address = (Inet6Address) address;
    if (isPreferIPV6Address()) {
        return Optional.ofNullable(normalizeV6Address(v6Address));
    }
}
if (isValidV4Address(address)) {
    return Optional.of(address);
}
return Optional.empty();
// 	检查地址是否可以访问（原理是发送一个ICMP协议简单请求）
addressOp.get().isReachable(100)
//6 上面获取的是InetAddress对象，再转成IP地址字符串，如果是IPv6可能包含%xxx,将%及后面的字符串删除即可
address.getHostAddress();
```



