---
title: springboot
date: 2021-07-13 23:42:53
categories:
tags:
---
1. spring中不同的bean之间的依赖，是通过spring来进行依赖注入的，bean之间交给spring来管理.
spring可以在xml配置文件中，将bean的依赖关系给阐述出来，让spring读取xml来识别谁依赖谁。

2. spring配置方式
a.基于xml的方式进行配置
在xml中描述bean之间的依赖关系
<bean id="xxx" class="yyy">
 <constructor-arg ref="zzz>
</bean>
b.基于java的配置
@Configuration
public class BbConfig {
    @Bean
    public Bb bb() {
        //依赖注入了aa这个bean
        // return new Bb(new Aa()); 这里bb中的Aa就变成了自己创建的对象了，而不是spring创建的
        return new Bb(aa());//这里aa()方法产生的就是spring给注入的bean
    }
    @Bean
    public Aa aa() {
        return new Aa();
    }
}

3. 工作方式
spring通过应用上下文（Application Context）装载bean的定义并把它们组装起来。Spring应用上
下文全权负责对象的创建和组装。Spring自带了多种应用上下文的实现，它们之间主要的区别
仅仅在于如何加载配置。

4. aop  
面向切面编程就是说，整个系统可以划分成各种不同功能的模块，每个模块负责自己的核心功能，
然后有些模块可以在不影响目标模块功能的前提下，作用到目标模块上，目标模块甚至都不知道自己被附加了这个模块，
这是一种无入侵式的功能绑定。
aop相对于调用一个抽象独立出来的方法而言，好处是对目标模块无注入影响，而公用方法，是需要嵌入到某个模块代码里面被调用的，
修改方法可能会影响目标模块。
aop的应用比如日志模块，事务模块，安全模块，可以用声明的方式作用到所要影响的组件上去。
aop的类是一个POJO类，是切面的同时前提也是一个spring bean，spring也可以像使用其他bean那样使用切面，比如依赖注入。

AOP原理
如何把切面织入到核心逻辑中？这正是AOP需要解决的问题。换句话说，如何对调用方法进行拦截，并在拦截前后进行安全检查、日志、事务等处理，就相当于完成了所有业务功能。

在Java平台上，对于AOP的织入，有3种方式：

编译期：在编译时，由编译器把切面调用编译进字节码，这种方式需要定义新的关键字并扩展编译器，AspectJ就扩展了Java编译器，使用关键字aspect来实现织入；
类加载器：在目标类被装载到JVM时，通过一个特殊的类加载器，对目标类的字节码重新“增强”；
运行期：目标对象和切面都是普通Java类，通过JVM的动态代理功能或者第三方库实现运行期动态织入。

最简单的方式是第三种，Spring的AOP实现就是基于JVM的动态代理。由于JVM的动态代理要求必须实现接口，如果一个普通类没有业务接口，就需要通过CGLIB或者Javassist这些第三方库实现。
AOP技术看上去比较神秘，但实际上，它本质就是一个动态代理，让我们把一些常用功能如权限检查、日志、事务等，从每个业务方法中剥离出来。
需要特别指出的是，AOP对于解决特定问题，例如事务管理非常有用，这是因为分散在各处的事务代码几乎是完全相同的，并且它们需要的参数（JDBC的Connection）也是固定的。另一些特定问题，如日志，就不那么容易实现，因为日志虽然简单，但打印日志的时候，经常需要捕获局部变量，如果使用AOP实现日志，我们只能输出固定格式的日志，因此，使用AOP时，必须适合特定的场景。
spring调用动态代理的对象，其实已经不是原来自己的对象了，而是被spring包装过的对象，和单纯调用原始的对象不一样


spring 通过CGLIB来创建一个被aop作用的类的代理类，核心就是先初始化一个原始的类对象，然后通过继承这个类创建一个代理类，
等实际运行时，表面上代码层是调用原始类对象的方法，实际上，真实运行调用的是被代理类的同名方法.


5. spring使用模板来消除重复样式的代码
比如使用JdbcTemplate，来避免各种重复的jdbc操作，包括创建连接，捕获sqlexception，关闭连接等

6. spring容器大致可以分为两类：
a.基本bean 工厂的容器,最基础
b.基于应用上下文的容器，基于bean工厂，分了各种xxxApplicationContext 上下文，每种上下文负责初始化各自范围内的bean
spring容器负责bean的整个生命周期，从出生到死亡。


7. spring对bean的操作有：自动扫描，自动装配，条件装配，占位符，spirng计算表达式，基于javaConfig，基于xml配置.


8. @Autowired作用在普通方法上，会在注入的时候调用一次该方法，如果方法中有实体参数，会对方法里面的参数进行装配，并调用一次该方法。
这个可以用来在自动注入的时候做一些初始化操作。

9. aop失效场景(原始类servcie，aop代理后的类$service，代理类相当于是包裹在service外的子类，通过同名方法，内部实际再调用原始对象的方法):
a.在同一个类service中，一个无aop作用的普通方法func1里调用另一个有aop作用的方法func2，此时func2的aop方法会失效
解释：当调用func1时，因为此方法非并没有aop作用，因此用的就是原始service对象，内部方法也是通过this.func2调用的，因此整个过程用的都是原始对象sevice
b.在同一个类service中，一个有aop作用的方法func1调用另一个有aop作用的方法func2，此时func2的aop方法会失效
解释：当调用func1时，因为此方法有aop作用，spring会生成一个增强代理类$service，于是通过$service来调用func1，而func2的调用是通过this.func2，也就是没走代理类$service，因此this就是原始sevice，并不是代理类$service，于是aop的效果就失效了。

解决方案:
a.将func1和func2放在两个不同的类中，然后需要依赖时，注入相关的依赖即可.
b.调用方法时，获取当前上下文的代理对象(通过AopContext.currentProxy())，然后通过代理对象进行方法调用,这种方案的前提是需要使用需要在AOP配置里暴露代理对象，在Spring Boot中可以通过注解@EnableAspectJAutoProxy(exposeProxy = true)进行配置。


10. spring boot
spring boot针对spring应用进行了变革，将各种繁琐的配置给简化，尽量的减少配置甚至无配置。
spring boot的自动配置特性，会探测pom.xml文件中的spirng-boot-starter-xxx.jar包，当加入相应的依赖时，就会自动配置出相应的bean出来使用，例如，加入spring-boot-starter-jdbc和com.h2database时，就会自动配置jdbcTemplate bean和 H2Database bean供使用.
@EnableAutoConfiguration启动自动配置.

11. spring boot项目内嵌tomcat和部署到独立的tomcat容器的启动类区别
a.当使用内嵌的tomcat时，不需要修改启动类，直接使用spring boot自动生成的启动类启动，此时包格式为jar包。
b.当使用独立的web容器运行项目时，例如jboss或者tomcat，则需要启动类继承SpringBootServletInitializer类，重写configure方法，保持main方法不动。
继承这个类的目的就是为了替代以往web.xml加载资源的方式。


12. spring事务
概念：spring本身没有事务，所使用的事务本质还是数据库本身的事务.
声明式开启方式：
启动类加注解@EnableTransactionManagement
方法或者类加注解@Transactional，推荐方法上使用，便于细粒度控制，防止事务造成的意外情况.



几个问题：
问题一：为啥要是用事务？
--为了保证一些数据库操作能同时成功和失败，因此需要放在一个数据库事务中来进行操作。

问题二：Spring是如何保证事务的？
--所谓的保证事务，目的就是让一些连续多次数据库操作在一个事务里面，因此通过共用一个数据库连接connection即可完成。
因此一个事务使用一个独立的connection，每个事务使用不同的connection。

问题三：spring中的service、dao等bean都是无状态的单例bean，单例的bean如何保证connection独立？
--操作数据库内部是通过一个线程来操作的，而connection是非线程安全的，同一时刻不能被多个线程共享，
因此每个线程需要持有独立的connection，并且互不相同、互不干扰。
所以我们使用事物同步管理器TransactionSynchronizationManager，实现事务的同步管理，利用ThreadLocal将数据库资源(connection)和当前事务绑定到一起，
使得事务传播时可以拿到正确的connection。TransactionSynchronizationManager 将 Dao、Service 类中影响线程安全的所有 “ 状态 ” 都统一抽取到该类中，
并用 ThreadLocal 进行封装，这样一来， Dao 和 Service就不用自己来保存一些事务状态了，从而就变成了线程安全的单例对象了。

参考资料：
https://cloud.tencent.com/developer/article/1497685
https://developer.aliyun.com/article/778661

13. spring mvc 流程：
   * http请求发送到前端控制器DispatcherServlet
   * 前端控制器请求处理器映射器HandlerMapping,返回对应的处理器适配器HandlerAdapter
   * 前端控制器请求适配器执行对应处理器Controller
   * controller处理器处理完后，如果是前后端一体的(@Controller)则返回ModelAndView，如果是restful风格的(@RestController)前后端分离，则返回相应的json或其他格式的字符串即可。
   * 前端控制器再将返回的结果response给客户端.如果是ModelAndView对象，则还需要进行视图解析，如果是Restful风格接口，则直接返回结果。

14. spring 核心技术 IOC AOP
   * IOC 控制反转，将new对象的工作交给spring 进行，spring启动时会扫描注解，根据依赖关系，自动的初始化bean，并放入对应的bean容器中，spring中好几种bean容器，因此启动时是多个容器同时进行初始化工作的；默认情况下，spring初始化出来的bean是单例的，也就是容器中只有一个bean对象，每个注入的地方都是复用同一个bean对象的。可以指定创建非单例的bean.
   * AOP 动态代理，可以实现面向切面编程，给一个bean动态的实现相应的方法。cglib,jdk方式两种不同的动态代理.   

15. spring中的事务注解，多数据源切换都是基于AOP实现的


16. mycat 
   * mycat 分页查询时会改写sql，比如select * from test limit M,N,会把sql改写成select * from test limit 0,M+N,将改写后的sql分发到各个节点执行，然后再放到mycat内存里进行汇总处理。因此mycat不适合M很大的分页，必须在sql中添加有效的过滤条件尽量的减少筛选出来的数量。