# Spring Security OAuth2

源码分析工作流程图参考 spring-security-oauth2-workflow.drawio。

旧版实现：[spring-security-oauth](https://github.com/spring-attic/spring-security-oauth)，和 Spring Security 没有什么关系，最后一个版本是2.5.2.RELEASE，现在项目已经废弃；

现在OAuth2功能迁移到了Spring Security oauth2 模块下，但是此模块没有提供认证服务器的实现；

认证服务器则是重新创建了个项目 [spring-authorization-server](https://github.com/spring-projects/spring-authorization-server)来维护，[文档](https://spring.io/projects/spring-authorization-server)。

新版的认证服务器不支持JDK8, 还是基于旧版做的测试。

其实看 Spring Security OAuth2 源码可以发现，OAuth2 里面涉及的HTTP接口、返回值、以及数据表（如果选择数据库实现ClientDetailsService）等都是写死的无法调整的（可能与项目的风格显得格格不入），需要注意实际项目有没有什么特殊定制需求，如果有的话最好还是自己按照 OAuth2 的标准自行实现，而不是用 Spring Security OAuth2 的封装。

案例参考： kwseeker/software-design-proposal/authentication/auth-java。
