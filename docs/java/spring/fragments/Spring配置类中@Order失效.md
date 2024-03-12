# Spring @Order 注解在配置类中失效

联动:  spring security 自动配置中会默认创建一个表单登录、BASIC认证的过滤器链的问题与解决。

关于 @Order 为何无效，测试了加载配置类上也无效，没有找到好的资料作出完美解释，有空了自己研究源码吧，Spring 主流程的源码已经看了一遍，只看这个问题应该不复杂。

贴下别人的分析：[踩坑！@Order失效。。。](https://blog.csdn.net/qq_34142184/article/details/126951618?spm=1001.2101.3001.6650.2&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-2-126951618-blog-125897498.235%5Ev43%5Epc_blog_bottom_relevance_base2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-2-126951618-blog-125897498.235%5Ev43%5Epc_blog_bottom_relevance_base2&utm_relevant_index=5)，可能对后续看源码有一点帮助。

