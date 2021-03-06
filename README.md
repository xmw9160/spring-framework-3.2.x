## 02-容器得基本实现
##### 1. 加载bean
```java
BeanFactory bf = new XmlBeanFactory(new ClassPathResource("beanFactoryTest.xml"));
MyTestBean bean = (MyTestBean)bf.getBean("myTestBean")
```

>*ConfigReader: 用于读取及验证配置文件  
*ReflectionUtil: 用于根据配置文件中的配置进行反射实例化  
*app: 完成整个逻辑的串联  

#### 2. 核心类介绍
* DefaultListableBeanFactory
> XmlBeanFactory继承自DefaultListableBeanFactory, 而DefaultListableBeanFactory是整个bean加载的核心部分, 是Spring注册及加载bean的默认实现, 而对于XmlBeanFactory与DefaultListableBeanFactory不同的地方其实是XmlBeanFactory中使用了自定义的Xml读取器XmlBeanDefinitionReader, 实现了个性化的BeanDefinitionReader读取, DefaultListableBeanFactory继承了AbstractAutowireCapableBeanFactory并实现了ConfigurableListableBeanFactory以及BeanDefinitionRegistry接口.
* `DefaultListableBeanFactory`类图

>
1.`AliasRegistry`：定义对alias的简单增删改等操作  
2.`SimpleAliasRegistry`：主要使用map作为alias的缓存，并对接口AliasRegistry进行实现  
3.`SigletonBeanRegistry`：定义对单例bean的注册及获取  
4.`BeanFactory`：定义获取bean及bean的各种属性  
5.`DefaultSingletonBeanRegistry`：对接口SigletonBeanRegistry各函数的实现  
6.`HierarchicalBeanFactory`：继承BeanFactory，也就是在BeanFactory定义的功能基础上增加了对parentFactory的支持  
7.`BeanDefinitionRegistry`：定义对BeanDefinition的各种增删改操作  
8.`FactoryBeanRegistrySupport`：在DefaultSingletonBeanRegistry基础上增加了对FactoryBean的特殊处理功能  
9.`ConfigurableBeanFactory`：提供配置Factory的各种方法  
10.`ListableBeanFactory`：根据各种条件获取bean的配置清单  
11.`AbstractBeanFactory`：综合FactoryBeanRegistrySupport和ConfigurableBeanFactory的功能  
12.`AutowireCapableBeanFactory`：提供创建bean、自动注入、初始化以及应用bean的后处理器  
13.`AbstractAtuowireCapableBeanFactory`：综合AbstractBeanFactory并对接口AutowireCapableBeanFactory进行实现  
14.`ConfigurableListableBeanFactory`：BeanFactory配置清单，指定忽略类型及接口等  
15.`DefaultListableBeanFactory`：综合上面所有功能，主要是对Bean注册后的处理  
* `XmlBeanFactory`对`DefaultListableBeanFactory`类进行了扩展，主要用于从XML文档中读取BeanDefinition，对于注册及获取Bean都是使用从父类DefaultListbaleBeanFactory继承的方法去实现，而唯独与父类不同的个性化方法就是增加了`XmlBeanDefinitionReader`类型的reader属性，在XmlBeanFactory中主要使用reader属性对资源文件进行读取和注册。  

* `DefaultListableBeanFactory` 完成对通过beanName的注册.
> 通过beanName注册: org.springframework.beans.factory.support.DefaultListableBeanFactory#registerBeanDefinition
* `SimpleAliasRegistry`完成对通过alias的注册
> 通过alias注册: org.springframework.core.SimpleAliasRegistry#registerAlias
### 03-默认标签得解析
#### 1. 对xml属性进行解析
* DefaultBeanDefinitionDocumentReader
    * 委托给BeanDefinitionParserDelegate对xml属性进行解析
##### 1. 解析beanDefinition
* 如果不存在beanName 那么根据Spring中提供的命名规则为当前bean生成对应的beanName
```java
BeanDefinitionReaderUtils.generateBeanName(beanDefinition, this.readerContext.getRegistry(), true)
```
#### 2. BeanDefinition
* BeanDefinition的三个主要子类
    * RootBeanDefinition
    * ChildBeanDefinition
    * GenericBeanDefinition(Srping 2.5+)

* 子元素meta
> org.springframework.core.AttributeAccessorSupport#attributes = linkedHashMap保存.
meta数据由BeanMetadataAttributeAccessor 对象保存
通过BeanDefinition的getAttribute(key)获取
```xml
<bean id="myTestBean" class="bean.MyTestBean">
    <meta key="testStr" value="aaaaaa" />
</bean>
```

* `DefaultListableBeanFactory` 完成对通过beanName的注册.
> 通过beanName注册: org.springframework.beans.factory.support.DefaultListableBeanFactory#registerBeanDefinition
* `SimpleAliasRegistry`完成对通过alias的注册
> 通过alias注册: org.springframework.core.SimpleAliasRegistry#registerAlias
### 05-bean得加载
```java
MyTest bean = (MyTestBean)bf.getBean("myTestBean");
```
*  `org.springframework.beans.factory.support.AbstractBeanFactory` 提供getBean的实现
    * `org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean` 具体实现
##### 5.1 FactoryBean的使用
* Spring为此提供了一个`org.springframework.beans.factory.FactoryBean`的工厂类接口, 用户可以通过实现该接口定制实例化bean的逻辑
```java
public class CarFactoryBean inplements FactoryBena<Car> {
    public Car getObject(){ return new Car()}
    public Class<Car> getObjectType(){return Car.class}
    public boolean isSingleton(){return false}
}
```
##### 5.2 缓存中获取单例bean
* 在创建单例bean的时候会存在依赖注入的情况, 而在创建依赖的时候为了避免循环依赖,Spring创建bean的原则是不等bean创建完成就会将创建bean的ObjectFactory提早曝光加入到缓存中, 一旦下一个bean创建时需要依赖上个bean, 则直接使用ObjectFactory.
* 获取单例bean
`org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String beanName, boolean allowEarlyReference)` allowEarlyReference允许早期依赖
##### 5.3 从bean的实例中获取对象
* `org.springframework.beans.factory.support.AbstractBeanFactory#getObjectForBeanInstance()` 用于检测当前bean是否是FactoryBean类型的bean, 如果是, 那么需要调用该bean对应的FactoryBean实例中的getObject()作为返回值.
    * `org.springframework.beans.factory.support.FactoryBeanRegistrySupport#getObjectFromFactoryBean()` 实际从Factory中解析bean.
        * `org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#postProcessObjectFromFactoryBean` 实现FactoryBean得后置处理
        * **Spring获取bean的规则**: 尽可能保证所有bean初始化后都会调用注册的`BeanPostProcessor`的`postProcessAfterInitialization`方法进行处理, 在实际开发的过程中大可以针对此特性设计自己的业务.
##### 5.4 获取单例
* 如果缓存中不存在已经加载的单例bean就需要从头开始bean的加载过程, Spring通过使用getSingleton的重载方法实现bean的加载过程. `org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton(java.lang.String, org.springframework.beans.factory.ObjectFactory<?>)`
##### 5.5 准备创建bean
*  `org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean(java.lang.String, org.springframework.beans.factory.support.RootBeanDefinition, java.lang.Object[])`
    * 处理override属性(lookup-meohtd、replaced-method) `org.springframework.beans.factory.support.AbstractBeanDefinition#prepareMethodOverrides`
* 在bean的实例化前会调用后置处理器的方法进行处理. `org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInstantiation()`
* 在bean的实例化后会调用后置处理器的方法进行处理.  `org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization()`
##### 5.6 循环依赖
* Spring容器循环依赖
> 1.构造器循环依赖
2.setter循环依赖
* 循环调用是无法解决的,除非有终结条件, 否则就是死循环,最终导致内存的溢出的错误.
* 5.6.1 Spring如何解决循环依赖的
* Spring容器将每一个正在创建的bean标识符放在一个"当前创建的bean池"中, bean标识符在创建过程中将一直保存在这个池中, 因此如果在创建bean过程中发现自己已经在"当前创建bean池"里时,  将抛出`BeanCurrentlyInCreationException`异常标识循环依赖; 而对于创建完毕的bean将从"当前创建bean池"中清除掉.
* Spring中将循环依赖的处理分为3中情况:
    * 构造器循环依赖
        * 标识通过构造器注入构成的循环依赖, 此依赖是无法解决的, 只能抛出`BeanCurrentlyInCreationException`异常标识循环依赖.
    * Setter循环依赖
        * 表示通过setter注入方式构成的循环依赖. 对于setter注入造成的循环依赖是**通过Spring容器提前暴露刚完成构造器注入但未完成其他步骤(如setter注入)的bean来完成的**, 而且**只能解决单例作用域的bean循环依赖**, `earlySingletonObjects = new HashMap<String, Object>(16)`.  对于`singleton`作用域的bean, 可以通过`setAllowCircularReferences(false)`来禁止循环应用.
        * 创建过程中所有状态容器存储类`org.springframework.beans.factory.support.DefaultSingletonBeanRegistry` 
    * prototype范围的依赖处理
        * 对于`prototype`作用域的bean, Spring容器无法完成依赖注入, 因为Spring容器不进行缓存`prototype`作用域的bena(都是`new`), 因此无法提前暴露一个创建中的bean.
    * [Spring中循环依赖的处理](https://www.iflym.com/index.php/code/201208280001.html)
##### 5.7 创建bean
* 真正创建bean: `org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#**doCreateBean(**(final String beanName, final RootBeanDefinition mbd, final Object[] args)`
* 5.7.1 创建bean实例
    * 将BeanDefinition转换为BeanWrapper：`org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBeanInstance()`
    * 依赖处理解决流程
`org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#addSingletonFactory()`

    * 构造函数自动注入 `org.springframework.beans.factory.support.ConstructorResolver#autowireConstructor()`
    * 无参构造函数自动注入 `org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean(java.lang.String, java.lang.Object, org.springframework.beans.factory.support.RootBeanDefinition)`
        * 实例化策略: `org.springframework.beans.factory.support.SimpleInstantiationStrategy#instantiate(org.springframework.beans.factory.support.RootBeanDefinition, java.lang.String, org.springframework.beans.factory.BeanFactory)`
        * **使用Cglib进行实例化**: `org.springframework.beans.factory.support.CglibSubclassingInstantiationStrategy.CglibSubclassCreator#instantiate`
* 5.7.3 属性注入
    * `org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean()`
        * 根据名字自动注入：
`org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#autowireByName()`
        * 根据类型自动注入：
`org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#autowireByType()`
* 5.7.4 初始化bean
`org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#initializeBean(java.lang.String, java.lang.Object, org.springframework.beans.factory.support.RootBeanDefinition)`
* 5.7.5 注册DisposableBean
`org.springframework.beans.factory.support.AbstractBeanFactory#registerDisposableBeanIfNecessary()`

### 06-容器的功能扩展
##### ApplicationContext和BeanFactory两者都是用于加载bean的, 但相比之下, ApplicationContext提供了更多的扩展功能, 简单一点就是ApplicationContext包含了BeanFactory的所有功能.
* 入口
```java
ApplicationContext bf = new ClassPathXmlApplicationContext("beanFactoryTest.xml")
```
##### 6.1 设置配置路径
* `org.springframework.context.support.AbstractRefreshableConfigApplicationContext#setConfigLocations(String location)`
##### 6.2 功能扩展
* refresh函数几乎包含了ApplicationContext中提供的所有功能
`org.springframework.context.support.AbstractApplicationContext#refresh()`
##### 6.3 环境准备
`org.springframework.context.support.AbstractApplicationContext#prepareRefresh()`
##### 6.4 加载BeanFactory
* `org.springframework.context.support.AbstractApplicationContext#obtainFreshBeanFactory()`  
        * 委托处理 `org.springframework.context.support.AbstractRefreshableApplicationContext#refreshBeanFactory()`
        * `DefaultListableBeanFactory`是容器的基础
* 6.4.1 定制BeanFactory
    * `org.springframework.context.support.AbstractRefreshableApplicationContext#customizeBeanFactory(DefaultListableBeanFactory beanFactory)`
* 6.4.2 加载BeanDefinition
    * `org.springframework.context.support.AbstractXmlApplicationContext#loadBeanDefinitions(org.springframework.beans.factory.support.DefaultListableBeanFactory)`
##### 6.5 功能扩展
* `org.springframework.context.support.AbstractApplicationContext#prepareBeanFactory()`
    * 增加SPEL语言的支持
    * 增加属性祖册编辑器
        * 使用自定义属性编辑器,通过继承`PropertyEditorSupport`, 重写setAsText方法.
        * 通过注册Spring自带的属性编辑器`CustomDateEditor`.
        * `org.springframework.beans.PropertyEditorRegistrySupport#createDefaultEditors()` 注册了一系列常用的属性编辑器.
##### 6.6 激活注册BeanFactoryPostProcessor
* `BeanFactoryPostProcessor`接口跟`BeanPostProcessor`类似, 可以对bean的定义(配置元数据)进行处理. 也就是说, Spring IoC容器允许`BeanFactoryPostProcessor`在容器实际实例化任何其他bean之前读取配置元数据, 并有可能修改它.
> 1.如果想改变实际的bean实例(例如从配置元数据创建的对象), 最好使用`BeanPostProcessor`.
2.`BeanFactoryPostProcessor`的作用域范围是容器级别的, 它只和你所使用的容器有关.
3.在Spring中存在对于`BeanFactoryPostProcessor`的典型应用, 比如`PropertyPlaceholderConfigurer`.
##### 6.7 初始化非延迟加载单例
* `org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization()`
* `ApplicationContext`实现的默认行为就是在启动时将所有单例bean提前进行实例化. 提前实例化意味着作为初始化过程的一部分, `ApplicationContext`实例会创建并配置所有的单例bean.
##### 6.8 finishRefresh
* 在Spring中提供了`LifeCycle`接口, `LifeCycle`中包含`start/stop`方法, 实现此接口后Spring会保证在启动的时候调用其`start`方法开始其生命周期, 并在`Spring`关闭的时候调用`stop`方法来结束生命周期, 通常用来配置后台程序, 在启动后一直运行(如对MQ进行轮询).
* `org.springframework.context.support.AbstractApplicationContext#finishRefresh()`

### 07-AOP
* Spring 2.0采用`@AspectJ`注解对POJO进行标注, 从而定义一个包含切点信息和增强横切逻辑的切面.
* `@AspectJ`注解使用AspectJ切点表达式语法进行切点定义, 可以通过切点函数、运算符、通配符等高级功能进行切点定义,拥有强大的连接点描述能力(方法级别).
##### 7.2 动态AOP自定义标签
* `org.springframework.aop.config.AopNamespaceHandler#init()`. 在解析配置文件的时候,一旦遇到`aspectj-autoproxy`注解时就会使用`AspectJAutoProxyBeanDefinitionParser`进行解析.
* 所有解析器, 因为是对`BeanDefinitionParser`接口的统一实现, 所以入口都是从parse函数开始的.
    * 注册`AnnotationAwareAspectJAutoProxyCreartor`
    * 对于`AOP`的实现, 基本上都是靠`AnnotationAwareAspectJAutoProxyCreator` 去完成, 它可以根据`@Point`注解定义的切点来自动代理相匹配的bean.
    
## 10-Spring事务
##### 10.1事务使用示例
* 默认情况下Spring中的事务处理只对`RuntimeException`方法进行回滚, 所以将RuntimeException替换成普通的Exception不会产生回滚效果.
##### 10.2 事务自定义标签
* `org.springframework.transaction.config.TxNamespaceHandler#init()`对`annotation-driven`进行处理.
* `org.springframework.transaction.config.AnnotationDrivenBeanDefinitionParser`对`<annotation-driven>`进行解析.
* 事务属性的获取规则
    * `org.springframework.transaction.interceptor.AbstractFallbackTransactionAttributeSource#computeTransactionAttribute()`
    * 如果方法中存在事务属性,则使用方法上的属性, 否则使用方法所在类上的属性, 如果方法所在类的属性上还是没有搜索到对应的事务属性, 那么再搜索接口中的方法, 再没有的话, 最后尝试搜索接口的类上面的生命.
##### 10.3 事务增强器
* `TransactionInterceptor` 支撑着整个事务功能的架构, 实现了事务的特性.
    * `TransactionInterceptor`继承自`MethodInteceptor`, `invoke()`事务处理重心.
* 创建事务
    * `org.springframework.transaction.interceptor.TransactionAspectSupport#createTransactionIfNecessary()`
    * 获取事务`org.springframework.transaction.support.AbstractPlatformTransactionManager#getTransaction`
    * `org.springframework.jdbc.datasource.DataSourceTransactionManager#doBegin`, 设置隔离级别, 更改自动提交... 
    * `PROPAGATION_REQUIRES_NEW` 表示当前方法必须在它自己的事务里面运行, 一个新的事务将被启动, 而如果有一个事务正在运行的话, 则在这个方法运行期间将被挂起. 而Spring中对此种传播方式的处理与新事务建立最大的不同点在于使用`suspend`方法将原事务挂起, 将信息挂起的目的是为了在当前事务执行完毕后再将原事务还原(CPU上下文切换).
    * `PROPAGATION_NESTED` 表示如果当前正有一个事务在运行中, 则该方法应该运行在一个嵌套的事务中, 别嵌套的事务可以独立于封装事务进行提交或者回滚(内嵌事务异常不会引起外部事务的回滚), 如果封装事务不存在, 行为就像`PROPAGATION_REQUIRES_NEW`. 对于嵌套式事务的处理, Spring中主要考虑了两种处理方式.
        * Spring中允许嵌入式事务的时候, 则首选设置保存点的方式作为异常处理的回滚.
        * 对于其他方式, 比如JTA无法使用保存点的方式, 那么处理方式与`PROPAGATION_REQUIRES_NEW`相同, 而一旦出现异常, 则由Spring的事务异常处理机制去完成后续操作. 
* 回滚处理
    * `org.springframework.transaction.interceptor.TransactionAspectSupport#completeTransactionAfterThrowing`
    * 回滚条件. 
        * 默认回滚条件(`DefaultTransactionAttribute#rollbackOn`): `ex instanceof RuntimeException || ex instanceof Error`. 使用`@Transaction(rollbackFor=Exception.class)` 对一般异常进行回滚, -->>`RuleBasedTransactionAttribute#rollbackOn`.
    * 回滚处理.
        * `org.springframework.transaction.support.AbstractPlatformTransactionManager#rollback()`
        * 回滚会触发`TransactionSynchronization`中对应的方法, **使用这一点可以实现事务正常/异常执行后进行其他处理**.
```java
TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
    @Override
    public void afterCommit() {
        execute();
    }
});
```
* 事务提交
    * `org.springframework.transaction.interceptor.TransactionAspectSupport#commitTransactionAfterReturning()`
    * `org.springframework.transaction.support.AbstractPlatformTransactionManager#commit`
    * 在事务异常处理规则的时候, 当某个事务既没有保存点又不是新事务, Spring对它的处理方式只是设置一个`回滚标识`. 这里的`回滚标识`应用场景为: 某个事务是另一个事务的嵌入事务, 但是这些事务又不在Spring的管理范围内, 或者无法设置保存点, 那么Spring会通过设置`回滚标识`的方式来禁止提交. 首先当某个嵌入事务发生回滚的时候会设置`回滚标识`, 而等到外部事务提交时, 一旦判断当前事务流被设置了`回滚标识`, 则由外部事务来统一进行整体事务的回滚. 所以, 当事务没有被异常捕获的时候也并不意味着一定会执行提交的过程.
    * 提交过程中需要考虑一下两个条件
        * 当事务状态中有保存点信息的话不会去提交事务
        * 当事务非新事务饿时候也不会去执行提交事务的操作
        * 以上条件主要考虑内嵌事务的情况, 对于内嵌事务, 在Spring中正常的处理方式是`将内嵌事务开始之前设置保存点, 一点内嵌事务出现异常便根据保存点信息进行回滚`, 但是如果`没有出现异常, 内嵌事务并不会单独提交, 而是根据事务流由最外层事务负责提交`, 所以如果当前存在保存点信息便不是最外层事务, 不做保存操作, 对于是否是新事务饿判断也是基于此考虑.
    * 最终提交由底层数据库连接执行, `org.springframework.jdbc.datasource.DataSourceTransactionManager#doCommit()`
    
### 11-SpringMVC
* Spring的MVC是基于Servlet功能实现的, 通过实现Servlet接口的DispatcherServlet来封装其核心功能实现, 通过将请求分派给处理程序, 同时带有可配置的处理程序映射、视图映射、本地语言、主题解析以及上载文件支持。
* Spring MVC的实现原理是通过Servlet拦截所有URL来达到控制的目的的，所以web.xm的配置是必须的。
##### 11.2 ContextLoaderListener
* `ContextLoaderListener`的作用就是穷Web容器时, 自动装配`ApplicationContext`的配置信息。因为它实现了`ServletContextListener`这个接口, 在`web.xml`配置这个监听器, 自动启动容器时, 就会默认执行她实现的方法, 使用`ServletContextListener`接口, 开发者能够在为客户端请求提供服务之前向`ServletContext`中添加任意对象. 这个对象在`ServletContext`启动的时候被初始化, 然后在`ServletContext`整个运行期间都是可见的.
* 每一个Web应用都有一个ServletContext与之想关联. `ServletContext`对象在应用启动时被创建, 在应用关闭的时候被销毁.  `ServletContext`在全局范围内有效, 类似于应用中的一个全局变量.
* 在`ServletContextListener`中的核心逻辑便是初始化`WebApplicationContext`实例并放至`ServletContext`中.
* ServletContextListener的使用
* Spring中的ContextLoaderListener
    * `ServletContext`启动后会调用`ServletContextLoader`的`contextInitialized()`方法.
    * `WebApplicationContext`继承自`ApplicationContext`, 在`ApplicationContext`的基础上又追加了一些特定于Web的操作及属性, 非常类似于通过编程方式使用Spring时使用的`ClassPathXMLApplicationContext`类提供的功能.
##### 11.3 DispatcherServlet
* `Servlet`是一个按照Java编写的程序, 此程序是基于Http协议的, 在服务器端运行的(如Tomcat), 是按照`Servelt`规范编写的一个Java类. 主要是处理客户端的请求并将其结果发送到客户端. `servlet`的生命周期是有`servelt`的容器来控制的, 它可以分为3个阶段: 初始化、运行和销毁：
    * 初始化阶段
        * servlet容器加载servlet类， 把servlet类的.class文件中的数据读取到内存中
        * servlet容器创建一个`servletConfig`对象。`ServletConfig`对象包含了servlet的初始化配置信息
        * servlet容器创建一个servlet对象
        * servlet容器调用servlet对象的init方法进行初始化
    * 运行阶段
        * 当`servlet`容器接收到一个请求时, servlet容器会针对这个请求创建servletRequest和servletResponse对象, 然后调用`service`方法, 并把这两个参数传递给service方法. service方法通过`servletRequest`对象获取请求信息,. 并处理该请求. 再通过`servletResponse`对象生成这个请求的响应结果. 然后销毁`servletRequest`和`servletResponse`对象. 不管这个请求是post提交还是get提交, 最终这个请求都会由`service`方法来处理.
    * 销毁阶段
        * 当Web应用被终止时, `servlet`容器会先调用servlet对象的`destory`方法, 然后再销毁`servlet`对象, 同时也会销毁与`servlet`对象相关联的`serviceConfig`对象.
* servlet的框架由两个java包组成: javax.servlet和javax.servlet.http.
    * `javax.servlet`包中定义了所有的servlet类都必须实现或扩展的通用接口和类.
    * `javax.servlet.http`包中定义了采用HTTP协议通信的`HttpServlet`类.
        * HTTP的请求方式包括delete、get、options、post、put和trace,在`HttpServlet`类中分别提供了相应的服务方法, 他们是doDelete()、doGet()、doOptions()、doPost()、doPut()和doTrace().
* Servlet的使用
* DispatcherServlet的初始化
    * `DispatcherServlet`的初始化过程主要是通过当前的servlet类型实例转换为`BeanWrapper`类型实例, 以便使用Spring提供的注入功能进行对应的属性注入. `DispatcherServlet`继承自`FrameworkServlet`, `FrameworkServlet`类中包含对应的同名属性, Spring会保证这些参数被注入到对应的值中.
    * `org.springframework.web.servlet.DispatcherServlet#initStrategies()` Initialize the strategy objects that this servlet uses.
##### 11.4 DispatcherServlet的逻辑处理
* `org.springframework.web.servlet.FrameworkServlet#processRequest()`
* `org.springframework.web.servlet.DispatcherServlet#doDispatch()` 函数中展示了Spring请求处理所涉及的主要逻辑.
* 11.4.1 MultipartContent 类型的request处理
    * `org.springframework.web.servlet.DispatcherServlet#checkMultipart()` 对于请求的处理, 首先考虑多MultipartContent类型的request进行处理.
* 11.4.2 根据request信息寻找对应的handler
    * `org.springframework.web.servlet.DispatcherServlet#getHandler()` --> `org.springframework.web.servlet.handler.AbstractHandlerMapping#getHandler()` 将hander封装成HandlerExecutionChain.
    * eg: `org.springframework.web.servlet.handler.SimpleUrlHandlerMapping`.  --> `org.springframework.web.servlet.handler.AbstractHandlerMapping#getHandler()` --> `org.springframework.web.servlet.handler.AbstractUrlHandlerMapping#getHandlerInternal()`. 链处理机制是Spring中非常常用的处理方式, 是AOP中重要的组成部分, 可以方便的对目标对象进行扩展及拦截.
    * `**org.springframework.web.servlet.handler.AbstractUrlHandlerMapping**` 需要debug
* 11.4.3 没有找到对应的Handler的错误处理
    * `org.springframework.web.servlet.DispatcherServlet#noHandlerFound()` 默认在response中返回404
* 11.4.4 根据当前的Handler寻找对应的HandlerAdapter
    * `org.springframework.web.servlet.DispatcherServlet#getHandlerAdapter()` 默认使用`org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter()`
* 11.4.5 缓存处理
    * `org.springframework.web.servlet.HandlerAdapter#getLastModified()` Spring判断是否过期, 通过判断请求的"If-Modified-Since"是否大于等于当前的getLastModified方法的时间戳, 如果是, 则认为没有修改.
* 11.4.6 HandlerInterceptor的处理
    * Servlet API定义的servlet过滤器可以在servlet处理每个Web请求的前后分别对它进行前置处理和后置处理.
    * SpringMVC允许通过处理拦截Web请求, 进行前置处理和后置处理. 处理拦截是在Spring的Web应用程序的上下文中配置的, 因此它们可以利用各种容器特性, 并引用容器中声明的任何bean. 
    * 处理拦截是针对特殊的处理程序映射进行注册的, 因此它只拦截通过这些处理程序映射的请求.
    * 每个处理拦截都必须实现`HandlerInterceptor`接口, 它包含三个需要实现的回调方法: 
        * `preHandler`: 处理请求前调用, 这个方法应该返回`true`, 允许`DispatcherServlet`继续处理请求. 否则, `DispatcherServlet`会认为这个方法已经处理了请求, 直接响应返回给用户.
        * `postHandler`: 处理请求后调用, 徐云访问返回的ModelAndView对象, 因此可以在它里面操作模型属性.
        * `afterComplation`: 所有请求处理完成之后被调用(如视图呈现之后). 
* 11.4.7 逻辑处理
    * `org.springframework.web.servlet.handler.SimpleServletHandlerAdapter#handle()` --> `org.springframework.web.servlet.mvc.AbstractController#handleRequest()`
* 11.4.8 异常视图的处理
    * 这里Spring主要的工作就是将逻辑引导至`HandlerExceptionResolver`类的resolveException方法处理.
    * `org.springframework.web.servlet.DispatcherServlet#processHandlerException()` 
* 11.4.9 根据视图跳转页面
    * 在逻辑处理的最后一定会涉及一个页面跳转的问题
    * `org.springframework.web.servlet.DispatcherServlet#processDispatchResult()` --> `render()`.