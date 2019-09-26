## Spring Framework源码学习[![Build Status](https://build.spring.io/plugins/servlet/wittified/build-status/SPR-PUBM)](https://build.spring.io/browse/SPR)

---
![](https://raw.githubusercontent.com/chenxingxing6/spring-framework/master/assert/1.jpg)

---
各个模块依赖关系：
![](https://raw.githubusercontent.com/chenxingxing6/spring-framework/master/assert/2.jpg)

---
##### 1.Spring IOC容器
> Spring IOC通过引入xml配置，由IOC容器来管理对象的生命周期,依赖关系等。
![](https://raw.githubusercontent.com/chenxingxing6/spring-framework/master/assert/3.jpg)
从图中可以看出，我们以前获取两个有依赖关系的对象，要用set方法，而用容器之后，它们之间的关系就由容器来管理。

---

> Spring容器的加载过程?
![](https://raw.githubusercontent.com/chenxingxing6/spring-framework/master/assert/4.jpg)
BeanDefinition是一个接口，用于属性承载，比如<bean>元素标签拥有class、scope、lazy-init等配置。bean的
定义方式有千千万万种，无论是何种标签，无论是何种资源定义，无论是何种容器，只要按照Spring的规范编写xml配
置文件，最终的bean定义内部表示都将转换为内部的唯一结构：BeanDefinition。当BeanDefinition注册完毕以后，
Spring的BeanFactory就可以随时根据需要进行实例化了。

---
##### 2.容器启动，进行实例化
实例化的工作会在容器启动后过AbstractApplicationContext中reflash方法自动进行。常用的ApplicationContext
实现类ClassPathXmlApplicationContext继承了AbstractApplicationContext类。  

AbstractApplicationContext里的reflash方法是Spring初始IOC容器一个非常重要的方法，不管你是ApplicationContext
哪个实现类，最终都会进入这个方法。

---

```html
该方法作用：创建加载Spring容器配置

@Override       
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // 1.设置和校验系统变量和环境变量的值  
        prepareRefresh();
        
        // 2.主要是创建beanFactory，同时加载配置文件.xml中的beanDefinition
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        
        // 3.给beanFactory注册一些标准组建，如ClassLoader，StandardEnvironment
        prepareBeanFactory(beanFactory);

        try {
            // 4.提供给子类实现一些postProcess的注册
            postProcessBeanFactory(beanFactory);

            // 5.调用所有BeanFactoryProcessor的postProcessBeanFactory()方法
            invokeBeanFactoryPostProcessors(beanFactory);

            // 6.注册BeanPostProcessor，BeanPostProcessor作用是用于拦截Bean的创建
            registerBeanPostProcessors(beanFactory);

            // 7.初始化消息Bean
            initMessageSource();

            // 8.初始化上下文的事件多播组建
            initApplicationEventMulticaster();

            // 9.ApplicationContext初始化一些特殊的bean
            onRefresh();

            // 10.注册事件监听器
            registerListeners();

            // 11.非延迟加载的单例Bean实例化
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
        }
        catch (BeansException ex) {
            destroyBeans();
            cancelRefresh(ex);
            throw ex;
        }
        finally {
            resetCommonCaches();
        }
    }
}
```

---
###### BeanFactory
```java
/**
 * 只对IOC容器的基本行为作了定义
 */
public interface BeanFactory {
	/**
	 * 用来引用一个实例，或把它和工厂产生的Bean区分开，就是说，如果一个FactoryBean的名字为a，那么，&a会得到那个Factory
	 */
	String FACTORY_BEAN_PREFIX = "&";

	Object getBean(String name) throws BeansException;

	<T> T getBean(String name, Class<T> requiredType) throws BeansException;

	Object getBean(String name, Object... args) throws BeansException;

	<T> T getBean(Class<T> requiredType) throws BeansException;

	<T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

	<T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);

	<T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);

	boolean containsBean(String name);

	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

	boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;

	boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

	@Nullable
	Class<?> getType(String name) throws NoSuchBeanDefinitionException;

	@Nullable
	Class<?> getType(String name, boolean allowFactoryBeanInit) throws NoSuchBeanDefinitionException;

	String[] getAliases(String name);
}
```
---
bean加在经过了很复杂的过程：protected<T> T doGetBean()   
> 1.转换对应的beanName,如果name="&aa",会去除&符合，如果<bean>带有alias属性，则取alias所最终的beanName。   
> 2.尝试从缓存中加载单例bean。如果加载不成功，会再次尝试从singletonFactories中加载。   
> 3.bean的实例化。假如我们需要对工厂bean进行处理，那么这里得到的其实是工厂bean 的初始状态。真正干活的则是getObjectForBeanInstance定义factory-method方法返回的bean。   
> 4.原型模式的依赖检查。如果A类有B的属性，B中有A的属性，则会产生循环依赖。   
> 5.将存储的Xml配置文件的GernericBeanDefinition转换为RootBeanDefinition。前文提到的用于承载属性的BeanDefinition有三个实现，GernericBeanDefinition，RootBeanDefinition和ChildBeanDefinition，如果父类bean不为空的话，这里会把所有的属性一并合并父类属性，因为后续所有的Bean都是针对RootBeanDefinition的。   
> 6.寻找依赖。在初始化一个bean的时候，会首先初始化这个bean所对应的依赖。   
> 7.根据不同的scope创建bean。scope属性默认是singleton，还有prototype、request等。  
> 8.类型转换。如果bean是个String，而requiredType传入了Integer，然后返回bean，加载结束。  


其中,最重要的步骤是(7),spring的常用特性都在那里实现.

---
##### 3.FactoryBean
> 1.BeanFactory ：BeanFactory定义了 IOC 容器的最基本形式。如果bean还比作是人，那么它可以理解成三界，三界
里有各种功能的人，它是一个容器，可以管理很多的人。

> 2.FactoryBean：一个Bean(但不普通)，用户可以通过实现该接口定制实例化。我们把bean比作是人，那么FactoryBean
则是女娲，首先它本身有人的特征，但它能够生产人。

  
---

##### 4.学习心得
1.主要是学习spring的思想和编码规范。Spring的代码很多，逻辑复杂，而spring的编程风格就是将复杂的逻辑进行分解，分成N个函数嵌套，每一层
都是对下一层的总结和概要。

2.真正干活的函数式以do开头的。

---
