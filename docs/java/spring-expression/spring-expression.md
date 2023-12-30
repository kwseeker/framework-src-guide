# [Spring Expression](https://docs.spring.io/spring-framework/reference/core/expressions.html)

Spring 表达式语言（简称“SpEL”）是一种强大的表达式语言，支持在运行时查询和操作对象图。该语言语法类似于Unified EL，但提供了附加功能，最显着的是方法调用和基本字符串模板功能。

> Unified EL 是 JSP 2.1 引入的特性， 参考：[Unified Expression Language](https://docs.oracle.com/javaee/5/tutorial/doc/bnahq.html)。

[SpEL功能](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref.html)：

- Literal expressions 字面量表达式
- Boolean and relational operators 布尔和关系运算符
- Regular expressions 正则表达式
- Class expressions 类表达式
- Accessing properties, arrays, lists, and maps 访问属性、数组、列表和映射
- Method invocation 方法调用
- Assignment 任务
- Calling constructors 调用构造方法
- Bean references Bean引用
- Array construction 数组构建
- Inline lists 内联List
- Inline maps 内联Map
- Ternary operator 三元运算符
- Variables 变量
- User-defined functions added to the context 定义添加到上下文的用户自定义函数
- reflective invocation of  Method 反射调用方法
- various cases of MethodHandle
- Collection projection
- Collection selection
- Templated expressions 模板化表达式



## 使用方法

案例参考：[Spring Expression Language Guide](https://www.baeldung.com/spring-expression-language)

测试案例：SpringBoot-Labs/spring-expression

### 注解中使用SpEL

SpEL表达式格式：`#{expression}`

SpEL表达式可以引用属性`${property.name}`，比如`#{${someProperty} + 2`。

```java
@Value("#{20 - 1}") // 19
private double subtract;

@Value("#{1 eq 1}") // true
private boolean equalAlphabetic;

@Value("#{400 > 300 || 150 < 100}") // true
private boolean or;

@Value("#{2 > 1 ? 'a' : 'b'}") // "a"
private String ternary;

@Value("#{'100fghdjf' matches '\\d+' }") // false
private boolean invalidNumericStringResult;

// 访问 List and Map 中的对象 
// 这里 carPark 是 Spring Bean 容器中的一个 Bean， carsByDriver 是 carPark 中的 Map 对象
@Value("#{carPark.carsByDriver['Driver1']}") // Model1

public MovieRecommender(CustomerPreferenceDao customerPreferenceDao,
                        @Value("#{systemProperties['user.country']}") String defaultLocale) {
    this.customerPreferenceDao = customerPreferenceDao;
    this.defaultLocale = defaultLocale;
}
```

### Spring Bean XML 配置文件中使用SpEL

同上，SpEL表达式格式：`#{expression}`

```xml
// Spring Bean XML 配置中也可以使用
<bean id="someCar" class="com.baeldung.spring.spel.Car">
   <property name="make" value="Some make"/>
   <property name="model" value="Some model"/>
   <property name="engine" value="#{engine}"/>
   <property name="horsePower" value="#{engine.horsePower}"/>
</bean>
```

### 编程式使用SpEL

Spring官方文档提供了大量编程式使用SpEL的案例，使用很简单，这里细节不赘述了。

+ `EvaluationContext`

  默认提供了两种实现：`SimpleEvaluationContext`、`StandardEvaluationContext`。

+ 类型转换

  当在表达式中使用泛型类型时，SpEL 会尝试使用Spring Core `org.springframework.core.convert.ConversionService`进行转换以维护它遇到的任何对象的类型正确性。

+ Parser配置

  通过`org.springframework.expression.spel.SpelParserConfiguration`控制某些表达式组件的行为。具体参考此类源码注释。

  从成员字段可以看到配置选项

  ```java
  //编译器模式
  private final SpelCompilerMode compilerMode;
  private final ClassLoader compilerClassLoader;
  //容器类自动扩容配置
  private final boolean autoGrowNullReferences;
  private final boolean autoGrowCollections;
  private final int maximumAutoGrowSize;
  private final int maximumExpressionLength;
  ```

  + SpEL编译器配置

    可以通过编译器配置在求值期间，让编译器生成一个体现运行时表达式行为的Java类，并使用该类来实现更快的表达式求值。

    编译器运行模式：
  
    + OFF（关闭，默认）
    + IMMEDIATE（立即编译）
    + MIXED（混合模式）

    在混合模式下，表达式会随着时间的推移在解释模式和编译模式之间默默切换。经过一定次数的解释运行后，它们会切换到编译的
    形式，如果编译的形式出现问题，表达式会自动切换回解释的形式。
  
    配置编译器运行模式的两种方式：
  
    + SpelParserConfiguration

        ```java
        SpelParserConfiguration config = new SpelParserConfiguration(SpelCompilerMode.IMMEDIATE,
            this.getClass().getClassLoader());
        SpelExpressionParser parser = new SpelExpressionParser(config);
        ```

    + 通过JVM系统属性或SpringProperties机制设置`spring.expression.compiler.mode`属性实现

      ```properties
      # 比如 application.properties 中设置 或者 通过 Properties 类设置
      spring.expression.compiler.mode=mixed
      ```
  
    编译器使用限制（目前无法编译以下类型的表达式）：
  
    + Expressions involving assignment 涉及赋值的表达式
  
    + Expressions relying on the conversion service 依赖于转换服务的表达式
  
    + Expressions using custom resolvers or accessors 使用自定义解析器或访问器的表达式
  
    + Expressions using selection or projection 使用selection或projection的表达式
  

### 语法参考

直接看官方文档和官方[单元测试](https://github.com/spring-projects/spring-framework/tree/main/spring-expression/src/test/java/org/springframework/expression/spel)吧，没有使用难度。

下面列举一些比较重要的用法。

+ [Literal Expressions](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/literal.html)

+ [Properties, Arrays, Lists, Maps, and Indexers](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/properties-arrays.html)

  如果访问一个根对象中不存在的属性，会调用根对象对应的getter setter 方法。

  ```java
  @Test
  public void test4() {
      StandardEvaluationContext context = new StandardEvaluationContext();
      Person arvin = new Person("Arvin", 18);
      context.setRootObject(arvin);
      
      //根对象 arvin 中不存在 student 属性，实际会调用 isStudent() 方法
      Expression expr = parser.parseExpression("student");
      Boolean isStudent = expr.getValue(context, Boolean.class);
      
      assertEquals(Boolean.FALSE, isStudent);
  }
  
  static class Person {
      private String name;
      private int age;
  
      public Person(String name, int age) {
          this.name = name;
          this.age = age;
      }
  
      //其他 getter setter 方法
  
      public boolean isStudent() {
          System.out.println("calling isStudent()");
          return false;
      }
  }
  ```

+ [Inline Lists](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/inline-lists.html)

+ [Inline Maps](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/inline-maps.html)

+ [Array Construction](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/array-construction.html)

+ [Methods](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/methods.html)

  调用静态方法：

  ```java
  String message = parser.parseExpression("T(String).format('Simple message: <%s>', 'Hello World')")
                  .getValue(context, String.class);
  ```

+ [Operators](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/operators.html)

  **赋值运算符**：

  ```java
  @Test
  public void testAssignmentOperators() {
      CustomValue cv = new CustomValue();
      parser.parseExpression("value").setValue(context, cv, 233);
      assertEquals(cv.getValue(), 233);
  
      CustomValue cv2 = new CustomValue();
      Integer value = parser.parseExpression("value = 666").getValue(context, cv2, Integer.class);
      assertEquals(value, 666);
  }
  ```

  > 对null的大于和小于比较遵循一个简单的规则: null被视为无(即不是零)。因此，任何其他值总是大于null (X > null总是为真)，任何其他值都不会小于0 (X < null总是为假)。

+ [Types](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/types.html)

  语法：`T(java.util.Date)`、`T(String)`。

+ [Constructors](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/constructors.html)

  > 注意：SpEL 并非不支持调用自定义类的构造方法，而是需要给出类的**完整全限定名**，只有`java.lang`包下的类可以使用类名。
  >
  > ```java
  > //java.lang包下的类可以直接使用类名
  > @Test
  > public void test4() {
  >     Expression exp = parser.parseExpression("new String('hello world').toUpperCase()");
  >     String message = exp.getValue(String.class);
  >     assertEquals("HELLO WORLD", message);
  > }
  > 
  > //除了“java.lang”包下的类其他所有类都需要全限定名
  > //这里没有使用全限定名会找不到类，会抛异常
  > @Test
  > public void test4_1() {
  >     assertThrows(SpelEvaluationException.class, () -> {
  >         Expression expr = parser.parseExpression(" new Person('Arvin', 18).name ");
  >         System.out.println("expr value: " + expr.getValue());
  >     });
  > }
  > 
  > //这里使用了全限定名，可以正常执行
  > @Test
  > public void test4_2() {
  >     Expression expr = parser.parseExpression(" new top.kwseeker.spring.expression.example01.HelloSpELTest.Person('Arvin', 18).name ");
  >     assertEquals("HELLO WORLD", expr.getValue());
  > }
  > ```

+ [Variables](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/variables.html)

  可以在评估上下文（`EvaluationContext`）中绑定变量，并在SpEL表达式中引用（`#VarName`）。

  解决一个值在多个地方被引用的问题。

  ```java
  Inventor tesla = new Inventor("Nikola Tesla", "Serbian");
  context.setVariable("newName", "Mike Tesla");
  //这里的getValue只是触发赋值语句处理，所以返回值可以忽略
  parser.parseExpression("name = #newName").getValue(context, tesla);
  ```

  还有两个默认定义的变量：

  + `#this`：当前评估对象（evaluation object），比如上面的 `name`。
  + `#root`：根上下文对象，比如上面的`tesla`。

  ```java
  List<Integer> primes = Arrays.asList(2, 3, 5, 7, 11, 13, 17);
  context.setVariable("primes", primes);
  
  // Select all prime numbers > 10 from the list (using selection ?{...}).
  // .?参考 Collection Selection
  String expression = "#primes.?[#this > 10]";
  List<Integer> primesGreaterThanTen = parser.parseExpression(expression).getValue(context, List.class);
  
  //下面获取的 #this 和 #root 是同一个对象
  Object thisObject = parser.parseExpression("#this").getValue(context, primes);
  Object rootObject = parser.parseExpression("#root").getValue(context, primes);
  ```

+ [Functions](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/functions.html)

  自定义函数，用于拓展 SpEL。该函数是通过`EvaluationContext`注册的。

+ [Bean References](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/bean-references.html)

  引用Bean。如果计算上下文已经配置了bean解析器，则可以通过使用 `@` 符号从表达式中查找bean；另外可以通过 `&` 符号获取工厂Bean。

+ [Ternary Operator (If-Then-Else)](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/operator-ternary.html)

  比如：`"false ? 'trueExp' : 'falseExp'"`。

+ [The Elvis Operator](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/operator-elvis.html)

  三元运算符的缩写。

  比如：`"name != null ? name : 'Unknown'"` 可以缩写为 `"name?:'Unknown'"`。

  除了空对象之外，SpEL Elvis操作符还检查空字符串，即`"name?:'Unknown'"` 准确来说对标的是 `"(name != null and !name.isEmpty()) ? name : 'Unknown'"`

+ [Safe Navigation Operator](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/operator-safe-navigation.html)

  安全导航操作符（`?.`）用于避免NullPointerException，它来自Groovy语言。通常，当您有一个对象的引用时，您可能需要在访问对象的方法或属性之前验证它不是空的。为了避免这种情况，安全导航操作符返回null而不是抛出异常。

  ```java
  //placeOfBirth 为null就直接返回null, 否则返回city字段值
  String city = parser.parseExpression("placeOfBirth?.city").getValue(context, tesla, String.class);
  ```

+ [Collection Selection](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/collection-selection.html)

  集合元素选择（`.?[selectionExpression]`）。

  ```java
  StandardEvaluationContext societyContext = new StandardEvaluationContext();
  IEEE ieee = new IEEE();
  ieee.Members[0]= tesla;
  societyContext.setRootObject(ieee);
  //从societyContext的rootObject（这里是ieee）的members字段（List<Inventor>）选择nationality == 'Serbian'的Inventor实例
  List<Inventor> list = (List<Inventor>) parser.parseExpression(
  		"members.?[nationality == 'Serbian']").getValue(societyContext);
  ```

+ [Collection Projection](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/collection-projection.html)

  将集合映射到新的集合（`.![projectionExpression]`）。

  ```java
  //相当于 Stream map().collect();
  List placesOfBirth = (List)parser.parseExpression("members.![placeOfBirth.city]");
  ```

+ [Expression templating](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/templating.html)

  表达式模板允许将字面量与一个或多个 evaluation 块混合。每个evaluation块用自定义的前缀和后缀字符分隔。常见的选择是使用`#{}`作为分隔符（ParserContext.TEMPLATE_EXPRESSION）。

  ```java
  String randomPhrase = parser.parseExpression(
  		"random number is #{T(java.lang.Math).random()}",
  		new TemplateParserContext()).getValue(String.class);
  ```



## SpEL工作原理

解析spring-expression模块源码，分析SpEL工作原理。



### SpEL在其他组件中的应用

单纯从 SpEL

#### Spring Security 权限校验

+ **WebSecurityConfiguration 中动态注入**

  @Value 通常用于注入外部化属性；但是也可以传入SpEL表达式，在运行时动态计算结果，然后注入被注解的参数，
  
  参考：https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/value-annotations.html
  
  ```java
  @Autowired(required = false)
  public void setFilterChainProxySecurityConfigurer(
      ObjectPostProcessor<Object> objectPostProcessor,
      //这里的意思是运行时执行Bean autowiredWebSecurityConfigurersIgnoreParents getWebSecurityConfigurers() 方法，
      //将结果注入 webSecurityConfigurers
      @Value("#{@autowiredWebSecurityConfigurersIgnoreParents.getWebSecurityConfigurers()}") List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers)
      throws Exception {
  	//...
  }
  
  @Bean
  public static AutowiredWebSecurityConfigurersIgnoreParents autowiredWebSecurityConfigurersIgnoreParents(
      ConfigurableListableBeanFactory beanFactory) {
      return new AutowiredWebSecurityConfigurersIgnoreParents(beanFactory);
  }
  ```
  
+  