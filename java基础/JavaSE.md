---
title: Javase复习
categories:
  - JVM
tags:
  - javaSE
abbrlink: 56288
date: 2022-9-20 19:54:20

---



# 接口与抽象类区别

当多个对象不能够抽象出共同的父类，但有共同的行为，这种情况下就需要将这种行为封装成接口。比如：登记这种行为：人，汽车，房子，都需要等级 ，但是这三类没有共同的特征。所以要单独将这一行为封装成一个接口。再如：电脑的USB接口，如果符合USB接口规范，就可以插进去，并且读取数据，但是iphone ,Mp3,U盘他们不是同一种类型。所以USB接口规范就要单独写了。



抽象类有构造方法，但不能被实例化，有实现方法

接口也有实现方法，但是要加上default



接口是一组规则的集合

接口和抽象类都不能被实例化



什么时候用抽象类，什么时候用接口：多个对象可以抽取共同的代码到父类，就可以用抽象类，如果抽象不了，但是多个类有共同的行为，那么使用接口定义定义相同规范。







# 泛型



 泛型的本质是为了参数化类型



在编译的时候进行参数校验，不符合的报错

一般有**泛型类**、**泛型接口**、**泛型方法**



泛型是什么：参数化类型，**在创建对象或调用方法的时候才明确下具体的类型**



在编译的时候，检测类型，如果不符合就保错。

泛型只在编译阶段有效







## 泛型的性能

泛型是一个编译时特性。在运行应用程序时，它几乎没有任何影响。





泛型不会影响运行时性能。但是，它们可能会对您的开发性能产生积极影响。多亏了泛型，您可以在编译时获得额外的检查，因此您能够更快地检测到某些错误。





# 多态



编译时多态和运行时多态





**[多态](https://so.csdn.net/so/search?q=多态&spm=1001.2101.3001.7020)是同一个行为具有多个不同表现形式或形态的能力**





多态的三个必要条件：继承、重写、**父类引用指向子类对象**





**对象类型和引用类型之间具有继承（类）/实现（接口）的关系；**

**引用类型变量发出的方法调用的到底是哪个类中的方法，必须在程序运行期间才能确定；**









# 面向对象与面向过程区别



- 面向过程把解决问题的过程拆成一个个方法，通过一个个方法的执行解决问题。
- 面向对象会先抽象出对象，然后用对象执行方法的方式解决问题。

**面向对象开发的程序一般更易维护、易复用、易扩展。**





# 深拷贝与浅拷贝



- **浅拷贝**：浅拷贝会在堆上创建一个新的对象（区别于引用拷贝的一点），不过，如果原对象内部的属性是引用类型的话，浅拷贝会直接复制内部对象的引用地址，也就是说拷贝对象和原对象共用同一个内部对象。
- **深拷贝** ：深拷贝会完全复制整个对象，包括这个对象所包含的内部对象。







# 注解















# 反射



**运行时动态获取类的信息**



**获取class对象的四种方式：**

- 直接通过类名.class获取
- Class.forName(类的全路径名)
- 对象实例.getClass()获取
- 类加载器.loadClass(类名)



**注解和动态代理用了反射**



你在类上加上@Component注解，Spring就帮你创建对象







# 动态代理



动态代理是代理模式的一种，代理模式有两种：静态代理和动态代理





## 静态代理与动态代理区别

静态代理需要自己写代理类，实现对应的接口，比较麻烦



某个对象提供一个代理，代理角色固定，以控制对这个对象的访问。 **代理类和委托类有共同的父类或父接口**



```java
// 目标对象
You you = new You();
// 构造代理角色同时传入真实角色
MarryCompanyProxy marryCompanyProxy = new MarryCompanyProxy(you);
// 通过代理对象调用目标对象中的方法
marryCompanyProxy.toMarry();
```



**静态代理对于代理的角色是固定的，如dao层有20个dao类，如果要对方法的访问权限进行代理，此时需要创建20个静态代理角色，引起类爆炸，无法满足生产上的需要，于是就催生了动态代理的思想。**



## jdk动态代理与cglib动态代理





**动态代理在创建代理对象上更加的灵活，动态代理类的字节码在程序运行时，由Java反射机制动态产生**

**通过反射机制在程序运行期，动态的为目标对象创建代理对象，无需程序员手动编写它的源代码。**





> 动态代理不仅简化了编程工作，而且提高了软件系统的可扩展性，因为反射机制可以生成任意类型的动态代理类。代理的行为可以代理**多个方法**，**即满足生产需要的同时又达到代码通用的目的**。



### jdk动态代理

**JDK动态代理的目标对象必须有接口实现**



代理类实现Invovationhandler接口，在invoke方法里面进行目标类方法的增强，比如在原方法前后打印日志，增强等等

**动态代理类基于java反射机制，根据传入的目标对象，生成代理对象对原方法进行增强**





**通过代理对象实现目标对象的功能**

（**创建代理类参数中传入目标类进行代理**）

```java
// 目标对象
You you = new You();
// 获取代理对象
JdkHandler jdkHandler = new JdkHandler(you);
Marry marry = (Marry) jdkHandler.getProxy();//通过代理类生成代理对象
// 通过代理对象调用目标对象中的方法
marry.toMarry();    


@Test
public void testDynamicLogProxy() {
        OrderServiceImpl orderService = new OrderServiceImpl();
        Class<?> clazz = orderService.getClass();
        DynamicLogProxy logProxyHandler = new DynamicLogProxy(orderService);
        //通过Proxy.newProxyInstance(类加载器, 接口s, 事务处理器Handler) 加载动态代理
        OrderService os = (OrderService) Proxy.newProxyInstance(clazz.getClassLoader(), clazz.getInterfaces(), logProxyHandler);
        os.reduceStock();
    }


```







```text
Java动态代理类中的invoke是怎么调用的？

答：在生成的动态代理类$Proxy0.class中，构造方法调用了父类Proxy.class的构造方法，给成员变量invocationHandler赋值，$Proxy0.class的static模块中创建了被代理类的方法，调用相应方法时方法体中调用了父类中的成员变量InvocationHandler的invoke()方法。
```





**JDK的动态代理依靠接口实现，如果有些类并没有接口实现，则不能使用JDK代理。**



### cglib动态代理

**利用ASM开源包，对代理对象类的class文件加载进来，通过修改其字节码生成子类来处理。**



JDK的动态代理机制**只能代理实现了接口的类**，**而不能实现接口的类就不能使用JDK的动态代理**，cglib是针对类来实现代理的，它的原理是对指定的**目标类生成一个子类**，**并覆盖其中方法实现增强**，但因为采用的是继承，**所以不能对final修饰的类进行代理**。



```
/*它是针对类实现代理的，类不用实现接口，CGlib对目标类产生一个子类，通过方法拦截技术拦截所有的方法调用
*/
public class DynamicCglibLogProxy implements MethodInterceptor 

```



cglib底层是用ASM框架，使用字节码技术生成代理类，你使用Java反射的效率要高

**CGLIB通过继承方式实现代理，在子类中采用方法拦截的技术拦截所有父类方法的调用并顺势织入横切逻辑。**









```java
得先导包。xml中导包



public class MyMethodInterceptor implements MethodInterceptor{

    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("这里是对目标类进行增强！！！");
        //注意这里的方法调用，不是用反射哦！！！
        Object object = proxy.invokeSuper(obj, args);
        return object;
    }  
}

 // 代理类class文件存入本地磁盘方便我们反编译查看源码
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "D:\\code");
        // 通过CGLIB动态代理获取代理对象的过程
        Enhancer enhancer = new Enhancer();
        // 设置enhancer对象的父类
        enhancer.setSuperclass(HelloService.class);
        // 设置enhancer的回调对象
        enhancer.setCallback(new MyMethodInterceptor());
        // 创建代理对象
        HelloService proxy= (HelloService)enhancer.create();
        // 通过代理对象调用目标方法
        proxy.sayHello();


```







#### cglib是怎么生成子类的

利用ASM开源包，对代理对象类的class文件加载进来，通过修改其字节码生成子类来处理。











### 两者区别



- **JDK动态代理实现接口，Cglib动态代理继承思想**
- **JDK动态代理（目标对象存在接口时）执行效率高于Ciglib**
- **如果目标对象有接口实现，选择JDK代理，如果没有接口实现选择Cglib代理**
- JDK动态代理的速度已经比CGLib动态代理的速度快很多



JDK 动态代理通过回调拦截方式，通过反射获取模板接口名字、内部方法以及参数，再原来的接口上修改，拼接，产生一个新的java代理对象（类似于mybatis的反序列化代码过程）











Cglib动态代理采用继承方式，底层基于asm字节码技术参数一个新的java代理对象





**jdk动态代理生成类速度快，调用慢，cglib生成类速度慢，但后续调用快**





# 序列化和反序列化



序列化将对象序列化为二进制数据流，反序列化将二进制数据流反序列化为原对象

**序列化的主要目的是通过网络传输对象或者说是将对象存储到文件系统、数据库、内存中。**





如果有些字段不想进行序列化，使用transient关键字修饰（`transient` 只能修饰变量，不能修饰类和方法。）





# IO流



根据功能分为输入流和输出流

根据数据的处理格式又分为字节流和字符流







# 设计模式



**单例、工厂**

















# JVM







## GC对象判断方法（死亡对象判断方法）

**引用计数法**

给对象中添加一个引用计数器：

- 每当有一个地方引用它，计数器就加 1；
- 当引用失效，计数器就减 1；
- 任何时候计数器为 0 的对象就是不可能再被使用的。



**可达性分析算法**





 **“GC Roots”** 的对象作为起点，从这些节点开始向下搜索，节点所走过的路径称为引用链，当一个对象到 GC Roots 没有任何引用链相连的话，则证明此对象是不可用的，需要被回收

**哪些对象可以作为 GC Roots 呢？**



- 虚拟机栈(栈帧中的本地变量表)中引用的对象
- 本地方法栈(Native 方法)中引用的对象
- 方法区中类静态属性引用的对象（类静态变量引用的对象）
- 方法区中常量引用的对象（常量引用的对象）
- 所有被同步锁持有的对象（synchronized持有的对象）





红黑树



arrlist安全版本 vector









# java注解



注解是怎样生效的

自定义注解

















# 红黑树



防止二叉排序树倾斜导致查找时间复杂度高为O(n)



红黑树是自平衡二叉排序树























# 海量数据去重

对整数，可以考虑用位图法，因为本题需要辨别不存在、出现一次和出现多次三个状态，一个二进位制0和1满足不了需求，

所以一个数用两个二进制位表示，00表示不存在，01表示出现一次，11出现多次，

则2.5亿整数需要的内存数为232 * 2bit = 1GB， 普通电脑即可满足，

即用232 * 2个二进制位来标识所有的整数，每两位依次记录整数1，2，3，4.

读取文件的整数，算出对应的二进制位数，如果是00，则改为01；是01，则改为11;

全部处理完后，取其中标记为11的位数，计算出其所代表的整数值就得到所求结果。

























# 项目





![image-20220917170332294](../../images/JavaSE/image-20220917170332294.png)













## 数据库表

- 商品类目表
- 商品详情表
- 订单表
- 订单详情表
- 卖家表











## 缓存





买家端看到的商品信息可以存在redis缓存里面



## 分布式锁



```java
public void orderProductMockDiffUser(String productId) {//保证这段代码，单线程的去访问
        long value = System.currentTimeMillis() + TIME;
        //加锁
         // redisLock.lock(productId,value);
        if (!redisLock.lock(productId,String.valueOf(value))){
            //如果加锁失败
            throw new SellException(101, "请求失败，当前访问人数过多，请稍后重试");
        }
        //1.查询该商品库存，为0则活动结束。
        int stockNum = stock.get(productId);
        if (stockNum == 0) {
            throw new SellException(100, "活动结束");
        } else {
            //2.下单(模拟不同用户openid不同)
            orders.put(KeyUtil.genUniqueKey(), productId);
            //3.减库存
            stockNum = stockNum - 1;
            try {
                //线程休眠，避免网络延时，io延时等，相当于线程等着这些都操作完再继续操作
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            stock.put(productId, stockNum);
        }


        //解锁
        redisLock.unlock(productId);


    }
```





