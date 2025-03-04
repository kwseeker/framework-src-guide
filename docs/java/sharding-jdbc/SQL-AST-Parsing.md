# SQL AST 解析&应用

参考: 

+ [斯坦福-编译原理](https://www.bilibili.com/video/av96207540/)

+ [编译器设计(使用教材: 编译器设计.第2版)](https://www.bilibili.com/video/BV1Bp4y1t7Cw?from=search&seid=10463750338319857506&spm_id_from=333.337.0.0)

+ 编译器设计(ENGINEERING A COMPILER).第2版

+ [700行手写编译器](https://github.com/archeryue/cpc)
+ [TinyC编译器实战](https://pandolia.net/tinyc/ch16_tinyc_compiler.html)

+ [代码与AST转换示例](https://astexplorer.net/)



## 编译原理 & AST

### 编译过程

全面而精炼的解释: [编译器基本流程](https://pandolia.net/tinyc/ch6_compiler_overview.html)

Java代码编译流程：

![](http://www.hollischuang.com/wp-content/uploads/2018/04/QQ20180414-203816.png)

前端编译：

源代码经过**分析器**（Parser）转换成中间代码：

中间经过**词法分析**（tokenize）、**语法分析**、**语义分析**;

词法分析从输入字符流中生成一个个**Token**，Token 中文翻译为标记，是一个字符串，也是构成源代码的最小单位；词法分析过程中，词法分析器还会对标记进行分类；

语法分析将语句（Statement）解析成**抽象语法树**（AST）或**LLVM IR**（LLVM Intermediate Representation）中间代码。

后端编译：

中间代码经过**优化器**（Optimizer）进行代码优化，如：常量替换、函数内联、死代码剔除等等；

最后经过**代码生成器**生成机器码（将中间代码结合各种处理器的指令集或JVM指令集翻译成对应的机器码）。

> LLVM 是模块化、可重用的编译器以及工具链技术的集合。可以复用编译器代码，降低编译器开发成本。
>
> ![](https://pic3.zhimg.com/v2-d6a2f944b910db85520f85743f39eaca_b.jpg)

### 语法分析 & AST

**关键概念**：

+ **Syntax** (语法)

+ **Lexeme** (词素)

  词素是构成词的要素，Token代表某一类的词素。

+ **Semantiz** (语义)

  语句的含义。

+ **Grammar** (文法)

  Syntax的集合。比如C4文法是C文法的一个子集。

**AST的作用**：

AST将代码的语义以树状结构呈现出来，方便单词或标记的定位，可用于代码检查、优化、改写，以及机器码生成。

**Antlr**

Antlr是可以根据输入自动生成语法树并可视化的显示出来的开源的跨语言语法分析器。

