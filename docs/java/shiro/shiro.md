# [Shiro](https://github.com/apache/shiro)

[Reference](https://shiro.apache.org/reference.html)

[samples](https://github.com/apache/shiro/tree/main/samples)

目标：

+ Shiro工作原理
+ 集成到Spring Boot的原理

Shiro 以简单易用著称。



## 功能

![](https://shiro.apache.org/images/ShiroFeatures.png)

+ 认证、授权以及授权后的访问控制
+ 会话管理
+ 加密
+ 支持在身份验证、访问控制或会话生命周期期间对事件监听
+ 支持聚合一到多个用户安全数据的数据源，并将其全部呈现为单个复合用户 “视图”
+ 支持SSO功能
+ 支持”Remember me“服务
+ "Run As" 允许用户假设以另一个用户的身份进行授权

+ ...



## 概述

### 架构

![](https://shiro.apache.org/images/ShiroArchitecture.png)

> Realm: 领域的意思，本质是存储用户认证信息的DAO，并提供认证和授权方法。

### 配置

支持两种配置：

+ 编程式配置
+ INI文件配置



## 原理

