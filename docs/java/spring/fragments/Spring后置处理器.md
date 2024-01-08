# Spring后置处理器

关于后置处理器调用时机参考 spring-context.drawio、spring-beans-factorybean.drawio 流程图，下面的接口调用时机流程图中都展示了。



## Spring后置处理器接口

+ **BeanFactoryPostProcessor**

  针对BeanFactory的后置处理器，可以通过这个后置处理器注册BeanDefinition、修改BeanDefinition、注册Bean、删除Bean等等，具体可以看`ConfigurableListableBeanFactory`接口的方法。

  ```java
  void postProcessBeanFactory(ConfigurableListableBeanFactory var1) throws BeansException;
  ```

  + **BeanDefinitionRegistryPostProcessor**

    拓展了通过`BeanDefinitionRegistry`注册、注销BeanDefinition的接口方法，具体功能可以看`BeanDefinitionRegistry`接口的方法。

    ```java
    void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry var1) throws BeansException;
    ```

+ **BeanPostProcessor**

  针对Bean的后置处理器。

  **Bean创建过程**分为3个阶段：**Bean实例化**、**属性填充**、**初始化**，初始化阶段会调用到此后置处理器方法。

  详细参考：spring-beans-factorybean.drawio。

  ```java
  //Bean初始化阶段调用初始化方法前会调用此方法（初始化方法即Bean init-method属性 或 @PostConstruct 制定的方法）
  @Nullable
  default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
  return bean;
  }
  //Bean初始化阶段调用初始化方法后会调用此方法
  @Nullable
  default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
  return bean;
  }
  ```

  + **DestructionAwareBeanPostProcessor**

    用于拓展在Bean销毁前执行额外处理。

    ```java
    //Bean销毁前执行
    void postProcessBeforeDestruction(Object var1, String var2) throws BeansException;
    //Bean是否需要执行销毁处理
    default boolean requiresDestruction(Object bean) {
     	return true;
    }
    ```

  + **InstantiationAwareBeanPostProcessor**

    注意这里是`Instantiation`，拓展针对Bean实例化前后处理。

    ```java
    //在Bean实例化之前执行
    @Nullable
    default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
        return null;
    }
    //在Bean实例化之后属性填充之前执行
    default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
        return true;
    }
    //Bean属性填充阶段，执行过@Autowired注入之后执行
    @Nullable
    default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) throws BeansException {
        return null;
    }
    //已经废弃，Bean属性填充阶段，执行过@Autowired注入之后执行
    @Deprecated
    @Nullable
    default PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException {
        return pvs;
    }
    ```

    + **SmartInstantiationAwareBeanPostProcessor**

      在`InstantiationAwareBeanPostProcessor`基础上拓展了3个方法。

      ```java
      //实例化bean之前预测bean的最终类型，比如解决循环依赖时，通过这个方法判断最终是创建的代理类型还是原类型
      @Nullable
      default Class<?> predictBeanType(Class<?> beanClass, String beanName) throws BeansException {
          return null;
      }
      //指定创建Bean使用的构造方法
      @Nullable
      default Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName)
          throws BeansException {
          return null;
      }
      //获取已经实例化但是还未完成初始化的Bean实例，这个方法在Spring解决循环依赖中有使用
      default Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
          return bean;
      }
      ```

  + **MergedBeanDefinitionPostProcessor** (spring-context)

    ```java
    //Bean实例化之后属性填充之前执行
    //从属性传参可以看到可以通过这个方法修改Bean的RootBeanDefinition，比如可以修改初始化方法，从而影响后面的初始化行为
    //AutowiredAnnotationBeanPostProcessor（用于处理@Autowire）的此方法用于从Bean定义中读取需要依赖注入的属性信息并缓存
    void postProcessMergedBeanDefinition(RootBeanDefinition var1, Class<?> var2, String var3);
    
    //重置Bean定义
    //AutowiredAnnotationBeanPostProcessor的此方法用于清除前面的缓存信息
    default void resetBeanDefinition(String beanName) {
    }
    ```



## Spring后置处理器的应用

### Dubbo中的应用

