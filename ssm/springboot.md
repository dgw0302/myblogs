---
title: springboot
categories:
 - 框架||项目
tags:
  - springboot
abbrlink: 46153
date: 2022-05-4
cover : https://www.w3cschool.cn/attachments/image/20170802/1501662448352296.png
---



# 自动装配原理



下面是主程序类，主入口类

```java
@SpringBootApplication
public class Application {

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
       

    }

}
```



下面是@SpringBootApplication注解的底层注解

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(
```



最主要的是这个@EnableAutoConfiguration注解：开启自动配置功能



而这个@EnableAutoConfiguration最重要的是下面这个@Import({AutoConfigurationImportSelector.class})类。

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import({AutoConfigurationImportSelector.class})
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};
}
```



 Spring的底层注解@import，给容器导入一个组件，导入的组件由AutoConfigurationPackages.Registrar.class；

 将主配置类（@SpringBootApplication）标注的类的所在包以及下面所有子包里面的所有组件扫描导Spring容器



```java
@Import(AutoConfigurationImportSelector.class)：给容器导入组件
```



 AutoConfigurationImportSelector：导入哪些组件的选择器；

： **SpringBoot不需要写配置文件的原因是，SpringBoot所有配置都是在启动的时候进行扫描并加载，SpringBoot的所有自动配置类都在Spring.factories里面，但是不一定会生效，生效前要判断条件是否成立，只要导入了对应的start，就有对应的启动器，有了启动器就能帮我们进行自动配置类**



 SpringFactoriesLoader.loadFactoryNames（EnableAutoConfiguration.class，beanClassLoader）

 （loadFactoryNames）方法，用类加载器获取一个资源



**loadFactoryNames会过滤掉我们不需要导入的自动配置类，也就是我们pom.xml没引入的start对应的自动配置类**



通过@Import导入的AutoConfigurationImportSelector类将jar包下面的META-INF/spring.factories文件的配置类（有需要）加载进容器中



## 总结：

Spring Boot启动的时候会通过@EnableAutoConfiguration注解找到META-INF/spring.factories配置文件中的所有自动配置类，并对其进行加载，这些自动配置类都是以AutoConfiguration结尾来命名的，它实际上就是一个JavaConfig形式的Spring容器配置类，通过@Bean导入到Spring容器中，以Properties结尾命名的类是和配置文件进行绑定的。它能通过这些以Properties结尾命名的类中取得在全局配置文件中配置的属性，我们可以通过修改配置文件对应的属性来修改自动配置的默认值，来完成自定义配置。



## 流程图

![image-20220504131429052](../../images/springboot/image-20220504131429052.png)

# 启动流程







## 主要流程

0.启动main方法开始



1.**初始化配置**：通过类加载器，（loadFactories）读取classpath下所有的spring.factories配置文件，创建一些初始配置对象；通知监听者应用程序启动开始，创建环境对象environment，用于读取环境配置 如 application.yml



2.**创建应用程序上下文**-createApplicationContext，创建 bean工厂对象



3.**刷新上下文（启动核心）**
3.1 配置工厂对象，包括上下文类加载器，对象发布处理器，beanFactoryPostProcessor
3.2 注册并实例化bean工厂发布处理器，并且调用这些处理器，对包扫描解析(主要是class文件)
3.3 注册并实例化bean发布处理器 beanPostProcessor
3.4 初始化一些与上下文有特别关系的bean对象（创建tomcat服务器）
3.5 实例化所有bean工厂缓存的bean对象（剩下的）
3.6 发布通知-通知上下文刷新完成（启动tomcat服务器）



4.**通知监听者-启动程序完成**



启动中，大部分对象都是BeanFactory对象通过反射创建





# **有时间完善。。。。。**

#### 















































