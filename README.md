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