---
title: spring
categories:
  - 框架
tags:
  - 框架||项目
abbrlink: 46153
date: 2022-05-3 
cover : https://www.w3cschool.cn/attachments/image/20170802/1501662448352296.png
---









**本文仅供本人浏览复习使用，可能有不对的地方和引用他人博客的地方，别来找我（很忙）。**



# spring脑图

https://www.processon.com/view/link/5f5075c763768959e2d109df#map



# IOC



脑图

https://www.processon.com/view/link/5f5e12c05653bb5e8a5d3723



加载流程图

![image-20220503182358108](../../images/spring/image-20220503182358108.png)





## BeanDefinition



顾名思义，用于定义bean信息的类，包含bean的class类型、构造参数、属性值等信息，每个bean对应一个BeanDefinition的实例。简化BeanDefinition仅包含bean的class类型。创建Bean得先扫描生成BeanDefinition放进map里面。





## BeanFactoryPostProcess和BeanPostProcessor

BeanFactoryPostProcess和BeanPostProcessor是spring框架中具有重量级地位的两个接口，理解了这两个接口的作用，基本就理解spring的核心原理了。为了降低理解难度分两个小节实现。



**BeanFactoryPostProcess**



BeanFactoryPostProcessor是spring提供的容器扩展机制，允许我们在bean实例化之前修改bean的定义信息即BeanDefinition的信息。其重要的实现类有PropertyPlaceholderConfigurer和CustomEditorConfigurer，PropertyPlaceholderConfigurer的作用是用properties文件的配置值替换xml文件中的占位符，CustomEditorConfigurer的作用是实现类型转换。BeanFactoryPostProcessor的实现比较简单，看单元测试BeanFactoryPostProcessorAndBeanPostProcessorTest#testBeanFactoryPostProcessor追下代码。



**BeanPostProcessor**



BeanPostProcessor也是spring提供的容器扩展机制，不同于BeanFactoryPostProcessor的是，BeanPostProcessor在bean实例化后依赖注入完毕后,初始化前后修改bean或替换bean。BeanPostProcessor是后面实现AOP的关键。

BeanPostProcessor的两个方法分别在bean执行初始化方法（后面实现）之前和之后执行，理解其实现重点看单元测试BeanFactoryPostProcessorAndBeanPostProcessorTest#testBeanPostProcessor和AbstractAutowireCapableBeanFactory#initializeBean方法，有些地方做了微调，可不必关注。



```java
public interface BeanPostProcessor {
	/**
	 * 在bean执行初始化方法之前执行此方法
	 */
	Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

	/**
	 * 在bean执行初始化方法之后执行此方法
	 */
	Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```





## 应用上下文ApplicationContext



应用上下文ApplicationContext是spring中较之于BeanFactory更为先进的IOC容器，ApplicationContext除了拥有BeanFactory的所有功能外，还支持特殊类型bean如上一节中的BeanFactoryPostProcessor和BeanPostProcessor的自动识别、资源加载、容器事件和监听器、国际化支持、单例bean自动初始化等。

BeanFactory是spring的基础设施，面向spring本身；而ApplicationContext面向spring的使用者，应用场合使用ApplicationContext。

具体实现查看AbstractApplicationContext#refresh方法即可。注意BeanFactoryPostProcessor和BeanPostProcessor的自定识别，这样就可以在xml文件中配置二者而不需要像上一节一样手动添加到容器中了。









## FactoryBean

FactoryBean是一种特殊的bean，当向容器获取该bean时，容器不是返回其本身，而是返回其FactoryBean#getObject方法的返回值，可通过编码方式定义复杂的bean。



实现逻辑比较简单，当容器发现bean为FactoryBean类型时，调用其getObject方法返回最终bean。当FactoryBean#isSingleton==true，将最终bean放进缓存中，下次从缓存中获取。改动点见AbstractBeanFactory#getBean。



下面的bean就是一个FactoryBean

```java
public class CarFactoryBean implements FactoryBean<Car> {

	private String brand;

	@Override
	public Car getObject() throws Exception {
		Car car = new Car();
		car.setBrand(brand);
		return car;
	}

	@Override
	public boolean isSingleton() {
		return true;
	}

	public void setBrand(String brand) {
		this.brand = brand;
	}
}
```



```java
public class FactoryBeanTest {

	@Test
	public void testFactoryBean() throws Exception {
		ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("classpath:factory-bean.xml");

		Car car = applicationContext.getBean("car", Car.class);
		applicationContext.getBean("car");
		assertThat(car.getBrand()).isEqualTo("porsche");
	}
}
```



## IOC加载流程



#### MainStarter

```java
public static void main(String[] args)   {

   // 加载spring上下文
   AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(MainConfig.class);

   Car car =  (Car) context.getBean("car");
   
   System.out.println(car.getName());


}
```



进入到AnnotationConfigApplicationContext构造方法



```java
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
   //调用构造函数
   this();
   //注册我们的配置类，将配置类注册到BeanDefinitionMap里面
   register(annotatedClasses);
   //IOC容器刷新接口
   refresh();
}
```



### this

这个this会干两件事

```java
public AnnotationConfigApplicationContext() {
   /**
    * 创建一个读取注解的Bean定义读取器
    * 什么是bean定义？BeanDefinition
    *
    * 完成了spring内部BeanDefinition的注册（主要是后置处理器）
    *注册了一系列的后置处理器，
    * 去注册了一个用来解析配置类的后（置处理器）
    */
   this.reader = new AnnotatedBeanDefinitionReader(this);

   /**
    * 创建BeanDefinition扫描器
    * 可以用来扫描包或者类，继而转换为bd
    *
    * spring默认的扫描包不是这个scanner对象
    * 而是自己new的一个ClassPathBeanDefinitionScanner
    * spring在执行工程后置处理器ConfigurationClassPostProcessor时，去扫描包时会new一个ClassPathBeanDefinitionScanner
    *
    * 这里的scanner仅仅是为了程序员可以手动调用AnnotationConfigApplicationContext对象的scan方法
    *
    */

   this.scanner = new ClassPathBeanDefinitionScanner(this);

}
```



AnnotatedBeanDefinitionReader注册了一系列后置处理器，最著名的就是ConfigurationClassPostProcessor，用来处理配置类的后置处理器







### register(annotatedClasses)



**最终调用doRegisterBean(beanClass, null, null, null)方法将配置类注册成bean定义**

主要是注册我们的配置类，将配置类注册到BeanDefinitionMap里面



### refresh()（重要）



先来看看源代码

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      //1:准备刷新上下文环境
      prepareRefresh();

      //2:获取告诉子类初始化Bean工厂  不同工厂不同实现
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

      //3:对bean工厂进行填充属性
      prepareBeanFactory(beanFactory);

      try {
         // 第四:留个子类去实现该接口
         postProcessBeanFactory(beanFactory);

         // 调用我们的bean工厂的后置处理器. 1. 会在此将class扫描成beanDefinition  2.bean工厂的后置处理器调用
         //
         invokeBeanFactoryPostProcessors(beanFactory);

         // 注册我们bean的后置处理器
         registerBeanPostProcessors(beanFactory);

         // 初始化国际化资源处理器.
         initMessageSource();

         // 创建事件多播器
         initApplicationEventMulticaster();

         // 这个方法同样也是留个子类实现的springboot也是从这个方法进行启动tomcat的.
         onRefresh();

         //把我们的事件监听器注册到多播器上
         registerListeners();

         // 实例化我们剩余的单实例bean.
         finishBeanFactoryInitialization(beanFactory);

         // 最后容器刷新 发布刷新事件(Spring cloud也是从这里启动的)
         finishRefresh();
      }

      catch (BeansException ex) {
         if (logger.isWarnEnabled()) {
            logger.warn("Exception  encountered during context initialization - " +
                  "cancelling refresh attempt: " + ex);
         }

         // Destroy already created singletons to avoid dangling resources.
         destroyBeans();

         // Reset 'active' flag.
         cancelRefresh(ex);

         // Propagate exception to caller.
         throw ex;
      }

      finally {
         // Reset common introspection caches in Spring's core, since we
         // might not ever need metadata for singleton beans anymore...
         resetCommonCaches();
      }
   }
}
```



#### invokeBeanFactoryPostProcessors

```
// 调用我们的bean工厂的后置处理器. 1. 会在此将class扫描成beanDefinition  2.bean工厂的后置处理器调用
//
invokeBeanFactoryPostProcessors(beanFactory);
```

解析配置类以及将那些被@Compoent和@Bean标记的类放进BeanDefinitionMap里面



实例化并调用所有注册的beanFactory后置处理器

最终调用invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors())方法

在invokeBeanFactoryPostProcessors方法中，Spring 会先去找到所有的实现了BeanDefinitionRegistryPostProcessor的 BeanFactory 后置处理器，然后先执行实现PriorityOrdered的，再执行实

现了Ordered的。其中最著名的就是ConfigurationClassPostProcessor，用来扫描被 @Component 和 @Bean 标记的对象，并注册其 BeanDefinition 元数据到 Spring 容器的 BeanDefinitionMap 中。

查找实现了BeanFactoryPostProcessor的后置处理器，并且执行后置处理器中的方法。



#### finishBeanFactoryInitialization

```
// 实例化我们剩余的单实例bean.
finishBeanFactoryInitialization(beanFactory);
```



实例化所有剩余的非懒加载单例,比如`invokeBeanFactoryPostProcessors`方法中根据各种注解解析出来的类，在这个时候都会被初始化。实例化的过程各种`BeanPostProcessor`开始起作用。

# bean的生命周期



![image-20220503173324528](../../images/spring/image-20220503173324528.png)













遍历BeanFefinetionMap

替换修改Bean的元信息

·

在bean实例完、属性填充后会检测Bean有没有实现Aware相关的接口，如果存在填充相关的资源



接下来是初始化，**初始化前后**还要处理BeanPostProcess子类的一些方法（该类实现类BeanPostProcess接口、）



然后就可以用了





























> （1）实例化Bean：
>
> 对于BeanFactory容器，当客户向容器请求一个尚未初始化的bean时，或初始化bean的时候需要注入另一个尚未初始化的依赖时，容器就会调用createBean进行实例化。对于ApplicationContext容器，当容器启动结束后，通过获取BeanDeﬁnition对象中的信息，实例化所有的bean。
>
> （2）设置对象属性（依赖注入）：
>
> 实例化后的对象被封装在BeanWrapper对象中，紧接着，Spring根据BeanDeﬁnition中的信息 以及 通过BeanWrapper提供的设置属性的接口完成依赖注入。
>
> （3）处理Aware接口：
>
> 接着，Spring会检测该对象是否实现了xxxAware接口，并将相关的xxxAware实例注入给Bean：
>
> ①如果这个Bean已经实现了BeanNameAware接口，会调用它实现的setBeanName(StringbeanId)方法，此处传递的就是Spring配置文件中Bean的id值；
>
> ②如果这个Bean已经实现了BeanFactoryAware接口，会调用它实现的setBeanFactory()方法，传递的是Spring工厂自身。
>
> ③如果这个Bean已经实现了ApplicationContextAware接口，会调用
>
> setApplicationContext(ApplicationContext)方法，传入Spring上下文；
>
> （4）BeanPostProcessor：
>
> 如果想对Bean进行一些自定义的处理，那么可以让Bean实现了BeanPostProcessor接口，那将会调用postProcessBeforeInitialization(Object obj, String s)方法。
>
> （5）InitializingBean 与 init-method：
>
> 如果Bean在Spring配置文件中配置了 init-method 属性，则会自动调用其配置的初始化方法。
>
> （6）如果这个Bean实现了BeanPostProcessor接口，将会调用
>
> postProcessAfterInitialization(Object obj, String s)方法；由于这个方法是在Bean初始化结束时调用的，所以可以被应用于内存或缓存技术；
>
> 以上几个步骤完成后，Bean就已经被正确创建了，之后就可以使用这个Bean了。
>
> （7）DisposableBean：
>
> 当Bean不再需要时，会经过清理阶段，如果Bean实现了DisposableBean这个接口，会调用其实现的destroy()方法；
>
> （8）destroy-method：
>
> 最后，如果这个Bean的Spring配置中配置了destroy-method属性，会自动调用其配置的销毁方法



















# 循环依赖

https://segmentfault.com/a/1190000023647227



![image-20220922225510329](../../images/spring/image-20220922225510329.png)









![image-20220922225957144](../../images/spring/image-20220922225957144.png)











### 三级缓存



```java
	// 从上至下 分表代表这“三级缓存”
	
	//一级缓存存放完整bean
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256); //一级缓存
	
	//二级缓存存放半成品bean（尚未填充属性）
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16); // 二级缓存
	
	//三级缓存存放的是函数接口（用于动态代理，用于生成代理对象，主要用于解耦）
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16); // 三级缓存
	
	
	
	/** Names of beans that are currently in creation. */
	// 这个缓存也十分重要：它表示bean创建过程中都会在里面呆着~
	// 它在Bean开始创建时放值，创建完成时会将其移出~
	private final Set<String> singletonsCurrentlyInCreation = Collections.newSetFromMap(new ConcurrentHashMap<>(16));
 
	/** Names of beans that have already been created at least once. */
	// 当这个Bean被创建完成后，会标记为这个 注意：这里是set集合 不会重复
	// 至少被创建了一次的  都会放进这里~~~~
	private final Set<String> alreadyCreated = Collections.newSetFromMap(new ConcurrentHashMap<>(256));

```



**一级缓存singletonObjects**



一级缓存存放的是已经初始化好的bean，即已经完成初始化好的注入对象的代理



**二级缓存earlySingletonObjects**



二级缓存存放的是还没有完全被初始化好的中间对象代理，即已经生成了bean但是这个bean还有部分成员对象还未被注入进来



**三级缓存singletonFactory**



存放的是函数接口。当触发getObject时候会生成代理对象(半成品)放进二级缓存











### getSingleton



```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        synchronized(this.singletonObjects) {
            singletonObject = this.earlySingletonObjects.get(beanName);
            if (singletonObject == null && allowEarlyReference) {
                ObjectFactory < ?>singletonFactory = this.singletonFactories.get(beanName);
                if (singletonFactory != null) {
                    singletonObject = singletonFactory.getObject();
                    this.earlySingletonObjects.put(beanName, singletonObject);
                    this.singletonFactories.remove(beanName);
                }
            }
        }
    }
    return singletonObject;
}
```







1. 先从一级缓存singletonObjects中去获取。（如果获取到就直接return）

   

2. 如果获取不到或者对象正在创建中（isSingletonCurrentlyInCreation()），那就再从二级缓存earlySingletonObjects中获取。（如果获取到就直接return）

   

3. 如果还是获取不到，且允许singletonFactories（allowEarlyReference=true）通过getObject()获取。就从三级缓存singletonFactory.getObject()获取。（如果获取到了就从singletonFactories中移除，并且放进earlySingletonObjects。其实也就是从三级缓存移动（是剪切、不是复制哦~）到了二级缓存）





```java
加入`singletonFactories`三级缓存的前提是执行了构造器，所以构造器的循环依赖没法解决
```







Spring容器会将每一个正在创建的Bean 标识符放在一个“当前创建Bean池”中，Bean标识符在创建过程中将一直保持在这个池中，而对于创建完毕的Bean将从当前创建Bean池中清除掉。

这个“当前创建Bean池”指的是上面提到的singletonsCurrentlyInCreation那个集合。











流程

### A B相互依赖

 

**简单实现的流程**

| getBean()          A                                         | getBean       B     |
| ------------------------------------------------------------ | ------------------- |
| ①先getSingleton(A)            ⑪getSingleton(A),从三级移到二级 | ⑥先getSingleton(B)  |
| ②加入正在创建池                ⑫返回此时二级缓存里面的A（虽然是半成品） | ⑦加入正在创建池子   |
| ③实例化A                                                     | ⑧实例化B            |
| ④加入三级缓存（A）                                           | ⑨加入三级缓存(A,B)  |
| ⑤getBean(B)递归                                              | ⑩getBean(A)递归回去 |
| ⑯getBean拿到B                                                | ⑬拿到半成品A        |
| ⑰A初始化                                                     | ⑭B初始化成功返回    |
| ⑱A创建完成                                                   | ⑮加入一级缓存       |
| ⑲加入一级缓存                                                |                     |





①先getSingleton(A)：第一次进来 ，一级没有且没有在创建，返回null.

⑪getSingleton(A)：第二次进来，一级没有但是在创建，并且二级没有，三级有并且调用getObject方法生成aop代理对象放入二级缓存，并且从三级缓存移出来

⑬拿到半成品A，虽然是半成品，但是拿到的实际是A的引用，当A初始化完成之后，就是成品A了



二级缓存其实就可以解决循环依赖



⑮⑲加入一级缓存的对象是从二级缓存取的完成了Aop代理后的对象（这个二级缓存也是三级缓存getObject后存进去的）。





**结合源码文字描述**


-  getBean 在容器（包括一二三级缓存）中获得A对象，没有则创建A对象；
- 初始化A对象，将A对象的 ObjectFactory 的 lamda 放入三级缓存；
- populateBean，填充属性B对象；
- getBean 在容器（包括一二三级缓存）中获得B对象，没有则创建B对象；
- 初始化B对象，将B对象的 ObjectFactory 的lambda放入三级 缓存；
- populateBean，填充属性A对象，在三级缓存中找到A对象（正在创建中的A对象）；
- 调用A的 ObjectFactory，创建半成品，将创建好的半成品放到二级缓存，并将三级缓存中的A移除；
- 取到 A 对象之后，给B对象中的 A 属性赋值，那么此时 B 对象则创建完成，放到一级缓存，三级缓存删除B；
- 此时再给 A 对象中的 B 对象赋值，将成品 A 放到一级缓存，将二级缓存中的 A 删除；
- 此时 A 对象创建完成，再创建 B 对象，但是在容器中直接拿到了 B 对象，则无需再次创建，完工。





下面是getBean的

### getBean简单实现



```java
package com.tuling.circulardependencies;


import org.springframework.beans.BeansException;
import org.springframework.beans.factory.annotation.Autowired;

import java.lang.reflect.Field;
import java.util.Collections;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

/***
 * @Author 徐庶   QQ:1092002729
 * @Slogan 致敬大师，致敬未来的你
 */
public class MainStart {

    public static void main(String[] args) throws Exception {

        // 已经加载循环依赖
        String beanName="com.tuling.circulardependencies.InstanceA";

        getBean(beanName);

        //ApplicationContext 已经加载spring容器

        InstanceA a= (InstanceA) getBean(beanName);

        a.say();
    }


    // 一级缓存 单例池   成熟态Bean
    private static Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    // 二级缓存   纯净态Bean (存储不完整的Bean用于解决循环依赖中多线程读取一级缓存的脏数据)
    // 所以当有了三级缓存后，它还一定要存在，  因为它要存储的 aop创建的动态代理对象,  不可能重复创建
    private static Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(256);

    // 三级缓存
    private static Map<String, ObjectFactory> factoryEarlySingletonObjects = new ConcurrentHashMap<>(256);


    // 标识当前是不是循环依赖   如果正在创建并且从一级缓存中没有拿到是不是说明是依赖
    private static Set<String> singletonsCurrentlyInCreation =
            Collections.newSetFromMap(new ConcurrentHashMap<>(16));

    /**
     * 创建Bean
     * @param beanName
     * @return
     */

    private static Object getBean(String beanName) throws Exception {


        Class<?> beanClass = Class.forName(beanName);


        Object bean=getSingleton(beanName);

        if(bean!=null){
            return bean;
        }


        // 开始创建Bean

        singletonsCurrentlyInCreation.add(beanName);

        // 1.实例化
        Object beanInstanc = beanClass.newInstance();

        ObjectFactory factory= () -> {

           JdkProxyBeanPostProcessor beanPostProcessor=new JdkProxyBeanPostProcessor();
            return beanPostProcessor.getEarlyBeanReference(bean,beanName);

        };


        factoryEarlySingletonObjects.put(beanName,factory);
        //  只是循环依赖才创建动态代理？   //创建动态代理

        // Spring 为了解决 aop下面循环依赖会在这个地方创建动态代理 Proxy.newProxyInstance
        // Spring 是不会将aop的代码跟ioc写在一起
        // 不能直接将Proxy存入二级缓存中
        // 是不是所有的Bean都存在循环依赖  当存在循环依赖才去调用aop的后置处理器创建动态代理

        // 存入二级缓存
       // earlySingletonObjects.put(beanName,beanInstanc);

        // 2.属性赋值 解析Autowired
        // 拿到所有的属性名
        Field[] declaredFields = beanClass.getDeclaredFields();


        // 循环所有属性
        for (Field declaredField : declaredFields) {
            // 从属性上拿到@Autowired
            Autowired annotation = declaredField.getAnnotation(Autowired.class);

            // 说明属性上面有@Autowired
            if(annotation!=null){

                Class<?> type = declaredField.getType();

                //com.tuling.circulardependencies.InstanceB
                getBean(type.getName());
            }

        }


        // 3.初始化 (省略）
        // 创建动态代理


      //由于递归完之后A还是原实例，所以要从二级缓存里面拿到动态代理后的bean,这个二级缓存是从三级缓存里面移动的
     //不要担心二级缓存里面A的属性没有填充，存放的是引用，到这一步引用指向的实例已经填充完毕
      beanInstanc = getSingleton(beanName);

        // 存入到一级缓存
        singletonObjects.put(beanName,beanInstanc);


      //remove 二级缓存和三级缓存
        return beanInstanc;
    }


    private  static Object getSingleton(String beanName){
        Object bean = singletonObjects.get(beanName);
        // 如果一级缓存没有拿到  是不是就说明当前是循环依赖创建
        if(bean==null && singletonsCurrentlyInCreation.contains(beanName)){
              // 调用bean的后置处理器创建动态代理
           //从二级缓存拿
            bean=earlySingletonObjects.get(beanName);
            if(bean==null){
               //从三级缓存拿
                ObjectFactory factory = factoryEarlySingletonObjects.get(beanName);

             if(factory != null) {
                bean = factory.getObject();
                earlySingletonObjects.put(beanName,bean);
             }
         }
        }
        return bean;
    }



    private  static  Object getEarlyBeanReference(String beanName, Object bean){

        JdkProxyBeanPostProcessor beanPostProcessor=new JdkProxyBeanPostProcessor();
         return beanPostProcessor.getEarlyBeanReference(bean,beanName);
    }

}
```





### 解决Bean不完整（多线程getBean(A)）



**双重检查**

**两个锁+一次if判断**

下面是线程安全的实现版本



```java
package com.tuling.circulardependencies;


import org.springframework.beans.BeansException;
import org.springframework.beans.factory.annotation.Autowired;

import java.lang.reflect.Field;
import java.util.Collections;
import java.util.Map;
import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

/***
 * @Author 徐庶   QQ:1092002729
 * @Slogan 致敬大师，致敬未来的你
 */
public class MainStart {

    public static void main(String[] args) throws Exception {

        // 已经加载循环依赖
        String beanName="com.tuling.circulardependencies.InstanceA";

        getBean(beanName);

        //ApplicationContext 已经加载spring容器

        InstanceA a= (InstanceA) getBean(beanName);

        a.say();
    }


    // 一级缓存 单例池   成熟态Bean
    private static Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    // 二级缓存   纯净态Bean (存储不完整的Bean用于解决循环依赖中多线程读取一级缓存的脏数据)
    // 所以当有了三级缓存后，它还一定要存在，  因为它要存储的 aop创建的动态代理对象,  不可能重复创建
    private static Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(256);

    // 三级缓存
    private static Map<String, ObjectFactory> factoryEarlySingletonObjects = new ConcurrentHashMap<>(256);


    // 标识当前是不是循环依赖   如果正在创建并且从一级缓存中没有拿到是不是说明是依赖
    private static Set<String> singletonsCurrentlyInCreation =
            Collections.newSetFromMap(new ConcurrentHashMap<>(16));

    /**
     * 创建Bean
     * @param beanName
     * @return
     */

    private static Object getBean(String beanName) throws Exception {


        Class<?> beanClass = Class.forName(beanName);


        Object bean=getSingleton(beanName);

        if(bean!=null){
            return bean;
        }
      Object beanInstanc = null;
        
      synchronized (earlySingletonObjects) {//第二个锁

         //解决多线程getBean(A)，判断
         if(earlySingletonObjects.containsKey(beanName)) {
            return singletonObjects.get(beanName);
         }

         // 开始创建Bean

         singletonsCurrentlyInCreation.add(beanName);

         // 1.实例化
         beanInstanc    = beanClass.newInstance();

         ObjectFactory factory= () -> {

            JdkProxyBeanPostProcessor beanPostProcessor=new JdkProxyBeanPostProcessor();
            return beanPostProcessor.getEarlyBeanReference(bean,beanName);

         };


         factoryEarlySingletonObjects.put(beanName,factory);
         Field[] declaredFields = beanClass.getDeclaredFields();


         // 循环所有属性
         for (Field declaredField : declaredFields) {
            // 从属性上拿到@Autowired
            Autowired annotation = declaredField.getAnnotation(Autowired.class);

            // 说明属性上面有@Autowired
            if(annotation!=null){

               Class<?> type = declaredField.getType();

               //com.tuling.circulardependencies.InstanceB
               getBean(type.getName());
            }

         }


         // 3.初始化 (省略）
         // 创建动态代理


         //由于递归完之后A还是原实例，所以要从二级缓存里面拿到动态代理后的bean,这个二级缓存是从三级缓存里面移动的

         beanInstanc = getSingleton(beanName);

         //把A存入到一级缓存
         singletonObjects.put(beanName,beanInstanc);

      }
      //remove 二级缓存和三级缓存


      return beanInstanc;
    }


    private  static Object getSingleton(String beanName){
        Object bean = singletonObjects.get(beanName);
        // 如果一级缓存没有拿到  是不是就说明当前是循环依赖创建
        if(bean==null && singletonsCurrentlyInCreation.contains(beanName)){

           synchronized (singletonObjects) {//第一个锁，锁的是一级缓存
            // 调用bean的后置处理器创建动态代理
            //从二级缓存拿
            bean=earlySingletonObjects.get(beanName);
            if(bean==null){
               //从三级缓存拿
               ObjectFactory factory = factoryEarlySingletonObjects.get(beanName);

               if(factory != null) {
                  bean = factory.getObject();
                  earlySingletonObjects.put(beanName,bean);
               }
            }
         }
        }
        return bean;
    }



    private  static  Object getEarlyBeanReference(String beanName, Object bean){

        JdkProxyBeanPostProcessor beanPostProcessor=new JdkProxyBeanPostProcessor();
         return beanPostProcessor.getEarlyBeanReference(bean,beanName);
    }

}
```



### 不需要三级缓存



**先说观点**

**当被代理的对象之间存在循环依赖问题的时候，需要使用三级缓存，如果没有代理对象的产生，只需要二级缓存即可解决问题。**



以下为转载，我现在还没有看明白为什么不需要三级缓存（呜呜呜。。。）

> 
>
> 在 Spring 容器中，同名的 bean 只有一个，
>
> 在创建对象的过程中，**不知道对象什么时候会被代理**，但是在创建对象的时候，要遵循标准的bean创建的流程，**也就是无论当前对象是否需要被代理，总会提前生成一个普通对象**
>
> 如果执行过程中发现需要创建代理对象怎么办？需要使用代理对象将原来的普通对象给覆盖掉
>
> **因为不知道当前对象什么时候会被引用，所以只能在第一次引用的时候来判断对象是否需要被代理**，所以通过 Lambda 表达式的方式进行回调，第一次需要的时候来判断是否需要被覆盖。
>
> 因为如果直接在二级缓存中创建好半成品对象之后，当被引用的时候，发现需要创建代理对象，此时会返回一个新的代理对象，**而代理对象和之前二级缓存中的对象不是同一个对象**
>
> 使用三级缓存，在需要 AOP 的时候，直接创建一个代理对象，然后直接将代理对象放到二级缓存中即可，以后该对象不会再变化，



**构造器注入没有办法解决循环依赖**， 想让构造器注入支持循环依赖，是不存在的



# AOP

## 基于JDK的动态代理

AopProxy是获取代理对象的抽象接口，JdkDynamicAopProxy的基于JDK动态代理的具体实现。TargetSource，被代理对象的封装。MethodInterceptor，方法拦截器，是AOP Alliance的"公民"，顾名思义，可以拦截方法，可在被代理执行的方法前后增加代理行为。

测试;

```java
public class DynamicProxyTest {

	@Test
	public void testJdkDynamicProxy() throws Exception {
		WorldService worldService = new WorldServiceImpl();

		AdvisedSupport advisedSupport = new AdvisedSupport();
		TargetSource targetSource = new TargetSource(worldService);
		WorldServiceInterceptor methodInterceptor = new WorldServiceInterceptor();
		MethodMatcher methodMatcher = new AspectJExpressionPointcut("execution(* org.springframework.test.service.WorldService.explode(..))").getMethodMatcher();
		advisedSupport.setTargetSource(targetSource);
		advisedSupport.setMethodInterceptor(methodInterceptor);
		advisedSupport.setMethodMatcher(methodMatcher);

		WorldService proxy = (WorldService) new JdkDynamicAopProxy(advisedSupport).getProxy();
		proxy.explode();
	}
}
```



## 基于CGLIB的动态代理

基于CGLIB的动态代理实现逻辑也比较简单，查看CglibAopProxy。与基于JDK的动态代理在运行期间为接口生成对象的代理对象不同，基于CGLIB的动态代理能在运行期间动态构建字节码的class文件，为类生成子类，因此被代理类不需要继承自任何接口。



利用ASM开源包，对代理对象类的class文件加载进来，通过修改其字节码生成子类来处理。



```java
public class DynamicProxyTest {

	private AdvisedSupport advisedSupport;

	@Before
	public void setup() {
		WorldService worldService = new WorldServiceImpl();

		advisedSupport = new AdvisedSupport();
		TargetSource targetSource = new TargetSource(worldService);
		WorldServiceInterceptor methodInterceptor = new WorldServiceInterceptor();
		MethodMatcher methodMatcher = new AspectJExpressionPointcut("execution(* org.springframework.test.service.WorldService.explode(..))").getMethodMatcher();
		advisedSupport.setTargetSource(targetSource);
		advisedSupport.setMethodInterceptor(methodInterceptor);
		advisedSupport.setMethodMatcher(methodMatcher);
	}

	@Test
	public void testCglibDynamicProxy() throws Exception {
		WorldService proxy = (WorldService) new CglibAopProxy(advisedSupport).getProxy();
		proxy.explode();
	}
}
```







## **Cglib和jdk动态代理的区别？**



jdk动态代理基于接口的实现类，基于反射调用目标类的代码，动态地将横切逻辑和业务逻辑编织在一起。



cglib是基于asm框架修改被代理类字节码生成子类进行代理。



以下来自引用



> Cglib和jdk动态代理的区别？
>
> 
>
> 1、Jdk动态代理：利用拦截器（必须实现InvocationHandler）加上反射机制生成一个代理接口的匿名类，在调用具体方法前调用InvokeHandler来处理
>
> 2、 Cglib动态代理：利用ASM框架，对代理对象类生成的class文件加载进来，通过修改其字节码生成子类来处理
>
> 
>
> JDK动态代理是面向接口的。
>
> CGLib动态代理是通过字节码底层继承要代理类来实现，因此如果被代理类被final关键字所修饰，会失败。
>
> 
>
> 什么时候用cglib什么时候用jdk动态代理？
>
> 1、目标对象生成了接口 默认用JDK动态代理
>
> 2、如果目标对象使用了接口，可以强制使用cglib
>
> 3、如果目标对象没有实现接口，必须采用cglib库，Spring会自动在JDK动态代理和cglib之间转换
>
> 
>
> JDK动态代理和cglib字节码生成的区别？
>
> 1、JDK动态代理只能对实现了接口的类生成代理，而不能针对类
>
> 2、Cglib是针对类实现代理，主要是对指定的类生成一个子类，覆盖其中的方法，并覆盖其中方法的增强，但是因为采用的是继承，所以该类或方法最好不要生成final，对于final类或方法，是无法继承的
>
> 
>
>  Cglib比JDK快？
>
> 1、cglib底层是ASM字节码生成框架，但是字节码技术生成代理类，在JDL1.6之前比使用java反射的效率要高
>
> 2、在jdk6之后逐步对JDK动态代理进行了优化，在调用次数比较少时效率高于cglib代理效率
>
> 3、只有在大量调用的时候cglib的效率高，但是在1.8的时候JDK的效率已高于cglib
>
> 4、Cglib不能对声明final的方法进行代理，因为cglib是动态生成代理对象，final关键字修饰的类不可变只能被引用不能被修改
>
> 
>
> Spring如何选择是用JDK还是cglib？
>
> 1、当bean实现接口时，会用JDK代理模式
>
> 2、当bean没有实现接口，用cglib实现
>
> 3、可以强制使用cglib（在spring配置中加入<aop:aspectj-autoproxy proxyt-target-class=”true”/>）













## AOP代理工厂



增加AOP代理工厂ProxyFactory，由AdvisedSupport#proxyTargetClass属性决定使用JDK动态代理还是CGLIB动态代理。









public class DynamicProxyTest {

```java
private AdvisedSupport advisedSupport;

@Before
public void setup() {
	WorldService worldService = new WorldServiceImpl();

	advisedSupport = new AdvisedSupport();
	TargetSource targetSource = new TargetSource(worldService);
	WorldServiceInterceptor methodInterceptor = new WorldServiceInterceptor();
	MethodMatcher methodMatcher = new AspectJExpressionPointcut("execution(* org.springframework.test.service.WorldService.explode(..))").getMethodMatcher();
	adviedSupport.setTargetSource(targetSource);
	advisedSupport.setMethodInterceptor(methodInterceptor);
	advisedSupport.setMethodMatcher(methodMatcher);
}

@Test
public void testProxyFactory() throws Exception {
	// 使用JDK动态代理
	advisedSupport.setProxyTargetClass(false);
	WorldService proxy = (WorldService) new ProxyFactory(advisedSupport).getProxy();
	proxy.explode();

	// 使用CGLIB动态代理
	advisedSupport.setProxyTargetClass(true);
	proxy = (WorldService) new ProxyFactory(advisedSupport).getProxy();
	proxy.explode();
}
}
```




## 动态代理融入bean生命周期





结合前面讲解的bean的生命周期，BeanPostProcessor处理阶段可以修改和替换bean，正好可以在此阶段返回代理对象替换原对象。不过我们引入一种特殊的BeanPostProcessor——InstantiationAwareBeanPostProcessor，如果InstantiationAwareBeanPostProcessor处理阶段返回代理对象，会导致短路，不会继续走原来的创建bean的流程，具体实现查看AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation。

DefaultAdvisorAutoProxyCreator是处理横切逻辑的织入返回代理对象的InstantiationAwareBeanPostProcessor实现类，当对象实例化时，生成代理对象并返回。



bean的生命周期如下：



![image-20220503182735315](../../images/spring/image-20220503182735315.png)













# 依赖注入

依赖注入的方式

- 1.xml注入
- 2.构造方法注入
- 3.setter注入
- 4.注解注入







# 代理对象什么时候生成(提前和非提前)





**非提前生成代理对象 是在属性填充populateBean完成之后，执行了initializeBean（初始化）方法的时候进行的动态代理。**



**提前动态代理，其实是在依赖注入的时候，也就是在populateBean属性填充方法内完成的。**



A对象与B对象



**A对象填充B对象（在这之前，刚实例化的A对象被已经加入到了三级缓存，但并没有提前生成代理对象），触发来B对象的实例化过程，然后，然后B对象去三级缓存中找A对象，并触发了A对象生成代理对象，并把A对象删除放进二级缓存。**







# 动态代理





## 静态代理与动态代理

https://zhuanlan.zhihu.com/p/159112639





通过实现被代理类实现的接口或者继承被代理类来动态生成代理类对象 在代理类对象重写的方法中实现对被代理类的增强？













传入被代理类的类型即可动态的生成代理类对象
