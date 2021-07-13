---
title: java
date: 2021-07-13 23:40:47
categories:
tags:
---
## java 并发
1. cpu,内存，磁盘 之间的效率问题衍生出并发问题：可见性 原子性 有序性
2. 解决并发: 就是解决上述衍生出来的三个问题
   * volatile: 解决可见性和有序性问题，可见性就涉及到java的内存模型：主内存和每个线程自己的工作内存。每个线程访问的变量都是从主内存拿取，然后放在线程自己的工作内存(比如cpu寄存器)中使用的，更新完变量值后，可以再写回主内存。被volatile修饰的变量，每次读都是从主内存读取最新，更新完后立即再写回主内存，并且修饰后变量的位置顺序不会被编译器给指令优化。
   * 锁：
        * 可以解决原子性问题。根据锁对并发性能的影响，产生了各种锁，synchronized,reentrantlock(可以配合condition使用,配合多个条件变量使用，然后可以通过wait将不同的线程放到不同的条件队列中等待唤醒,即等待通知机制)，readwritelock(内部分为读锁和写锁,读读ok,读写阻塞，写写阻塞，应用于读多写少)
        * synchronized锁可重入，不可中断，非公平锁
        * Lock的实现类锁可重入，可中断，也可以设置公平锁或者非公平锁,默认非公平锁
   * 无锁方式解决并发安全问题：
      * final 修饰变量
      * 使用ThreadLocal不共享变量
      * 原子类中使用CAS(compare and swap比较并交换，内部用到了自旋)
<!-- more -->
## java 容器
1. list:非线程安全(ArrayList LinkedList),线程安全(Vectory,使用Colletcions.synchronizedList(new ArrayList<>()),CopyOnWriteArrayList)
   * CopyOnWriteArrayList: 是一种Copy-On-Write(COW)模式，即写时复制，适用于对读有高性能要求的场景，因为写时通过加锁会复制内部数组里面的元素，然后用新数字替换老数组，内部用了volatile。
2. map: 
   * HashTable：
     * 内部数据结构：数组+链表
     * 线程安全的，使用了synchronized，效率低
     * 不能放null的key
     * 初始容量为11
   * HashMap :
     * 内部的数据结构时用数组+链表，根据hash值来定位在数组中的位置，同样的hash值再放入链表中，1.8用的尾插法，1.7用的头插法
     * 初始长度为16，默认的加载因子为0.75(Constructs an empty HashMap with the default initial capacity  (16) and the default load factor (0.75));并且数组的容量扩充或者初始化时 必须时2的幂次方
     * 从hash值映射到数组位置使用的时高效的位运算index=hashcode(key) & (length-1);之所以数组的长度为16或者2的幂次方，是因为这种情况下leng-1的二进制所有位都是1(比如15是1111)，因此index的值就等同于hashcode的后几位值，只要key本身均匀，那么hash算法的结果就是均匀的。
     * hashmap是非同步的，因此并发情况下是非线程安全的：1.多线程同时插入数据时 2.resize时可能造成死循环
     * 可以放null值的key和value
     * 如果链表的长度超过阈值8，就会转换成红黑树(搜索性能高),如果链表长度小于6，就会把红黑树转回链表.红黑树是为了解决链表查询深度问题，单链表过长时查询性能低
     * 如果容量满了(16*0.75),就会扩容2倍，然后重新插入数据
     * 7和8里面的区别：8引入了红黑树，7没有；链表中插入新节点8时尾插法，7时头插法
   * ConcurrentHashMap：
     * 基于hashmap,但是线程安全，性能比hashtable高 
     * 7和8的区别:
       * 7中基于分段锁，先分成一个一个的segment，然后segment里面再放元素,锁会锁指定的segment，其他的segment可以正常读取 
       * 8中抛弃了分段锁，采用cas+synchronize。如果元素第一次进入才用CAS方式插入，如果已经有链表了，则采用synchronized锁住表头进入插入.
   * TreeMap: 是对红黑树的一种实现，插入元素时可以对元素根据key进行排序。

## java 线程
1. 通过Thread类初始化一个线程
2. 通过直接实现runnable接口，再以参数形式传入到Thread中
3. 可以通过callable+future/futertask配合，放入Thread中执行，可以获取线程执行的状态和结果
4. 线程有sleep，wait方法，可以通过interrupt()方法终止线程，接收到终止信号的处于阻塞中的线程会发生InterruptedException,这些异常需要开发人员介入处理
5. 异步编程：
   * 可以使用CompletableFuture进行线程之间的协调通信，分为异步和通过方式，链式调用协调执行。
   * 可以使用countdownlatch进行多线程步调一致,latch对象会阻塞主线程等待子线程执行完。
   * 可以使用CompletionService实现批量的执行异步任务,同时发起多个异步任务，然后哪个先到达就先得到哪个的执行结果。
6. 创建线程池的方式，不能使用newFixedThreadPool(int nThreads)，因为内部的任务队列时无界的，可能会产生oom，必须通过ThreadPoolExecutor自定义线程池，然后自己指定核心数，最大数，线程空闲时间，时间单位，放置任务的阻塞队列，创建线程工厂，拒绝任务策略.有界的阻塞队列ArrayBlockingQueue和LinkedBlockingQueue，必须指定队列大小；丢弃策略有4种:直接丢弃并抛异常，直接丢弃不抛异常，抛弃最老的任务，使用调用线程执行任务.

## java中的动态代理
1. 静态代理:
   * A接口，B实现A，人工再创建一个C实现A，将B注入C，然后通过调用C中的同名方法来实现额外的扩充功能
   * 缺点就是，需要人工创建好每个需要的代理类
2. 动态代理：
   * 可以利用java反射机制，动态的生成一个代理类，不需要提前写好
   * java本身的动态代理是基于接口的动态代理(JDK代理)，而spring中的AOP引入了cglib代理(比如事务注解修饰的类或者方法)，可以实现不是接口也可以动态代理，是通过动态生成一个子类来实现代理类方法的,spring 会在需要的时候创建好动态类，插入特定的功能.
   * spring中动态代理的选择方式:
     * 如果目标对象实现了接口，则默认情况下采用jdk代理，也可以通过spring配置指定强制都采用cglib代理
     * 如果目标对象没有实现接口，则采用cglib代理

## jvm 
1. jvm内存模型:jdk1.8为标准
   * 内部结构分为两类：线程共享和线程私有
   * 线程共享：堆和方法区(元空间)
     * 堆java heap：存放对象的，GC作用的地方，是jvm中占用内存最大的地方,里面
     * 方法区(非堆Non-Heap)：存放类信息、常量、静态变量。1.8使用元空间来替代方法区,逻辑上也叫永久代
   * 线程私有：栈(jvm栈和本地方法栈)和程序计数器
     * 栈：分为jvm栈和本地方法栈，jvm用于执行java方法的，本地方法栈则用于执行native方法的
     * 程序计数器：用于当前线程所执行的字节码的行号指示器.
   * 参考：https://www.nowcoder.com/discuss/151138?type=1
2. GC算法
   * GC目前是采用分代回收算法，氛围新生代，老年代，永久代，不同代有具体的不同回收算法，新生代采用复制算法，老年代就需要采用标记清除算法
   * jvm提供的年轻代回收算法属于复制算法，CMS(8默认)、G1(9默认，8可用)，ZGC(11)属于标记清除算法。
   * 垃圾收集器:
       * CMS收集器，并发标记清理，先把所有活动的对象标记出来，然后把没有被标记的对象统一清除掉。但是它有两个问题，一是效率问题，两个过程的效率都不高。二是空间问题，清除之后会产生大量不连续的内存,产生空间碎片 
       * G1收集器优化了CMS，减少停顿，没有了物理上的年轻代，老年代，只有一些逻辑上的划分，使用一些非连续的区域来表示。
   * 参考：https://www.cnblogs.com/ityouknow/p/5614961.html
   * GC root：
       * 垃圾收集器是通过可达性分析算法来判断一个对象是否存活的，可达性分析就会以GC root对象为起点，一直往下寻找下一个节点，直到遍历完所有节点。如果相关对象不在任意一个以GC root的为起点的引用链，那么就判定这个对象为垃圾对象。
       * gc root有哪些：栈(jvm栈和本地方法栈)中的引用对象，方法区中的静态属性引用的对象(public static Test s)和方法区中常量的引用的对象(public static final Test s)
3. jvm参数和命令:
   * 参数:
      * -Xmx：指定对最大值
      * -Xms：指定堆初始值,通常这两个值大小设置一样的
      * -XX:MetaspaceSize=1024m,设置元空间大小
   * 命令:
      * jstack:显示java虚拟机当前的线程快照
      * jmap:显示java进程内存分配情况


## http
1. 由于http是无状态的，因此如果服务端需要记住用户的状态，就需要通过session来识别用户即session id. session可以存放在内存，数据库，文件中都可。
2. 跟踪session是通过cookie实现的，cookie中记录了session id，每次http发送请求时都会加上cookie信息给服务端，服务端存储了session文件，类似一个map，根据seesion id来寻找对应的会话.
3. http，session，cookie三种都是独立的东西，http是通过session和cookie来实现有状态的



## redis
1. redis数据类型
   * string：key-value格式，命令：set,get,del 举例：set age 23，set name "dpp",get age,del name, 不存在的key返回nil
   * list: key-value格式，value为列表,命令：rpush mylist e1 将e1放到mylist列表的右端，lrange mylist 0 -1 ,0为起始未知，-1为结束未知，查询出所有的元素,lpop mylist 从mylist列表左边弹出一个元素
   * hash :key-value 格式,value内存也是key1-value1格式，hset myhash field1 value1,hmset myhash field1 value1 field2 value2 同时设置多个k-v，可以用于存对象类型数据
   * set:key-value格式,无序集合并且可以去重，内存存储的是string类型数据，sadd myset s1
   * zset:key-value格式,有序集合并且可以去重,每个元素有对应的分数
2. redis分布式锁:
   * setnx 
3. redis集群codis
      * 介绍：是redis的一个分布式方案,为了严格的数据一致性，slave是异步复制master，存在延迟和数据丢失风险，不支持读写分离；基于codis-proxy来访问server group,每个group 里面分为master和slave,分布式逻辑写在proxy，底层存储在redis，数据的分布状态存储在zk
      * 现在生产集群节点是5*2的存储集群
      * codis通过presharding方式，先分出1024个slot，然后再将slot分配到多个redis组中，数据根据key计算到唯一的slot id，迁移数据也是将整个slot迁移走,hash算法是crc32(key)%1024
      * Redis的主从切换是通过codis-ha在zk上遍历各个server group的master判断存活情况，来决定是否发起提升新master的命令。
      * 一致性hash算法：
         * 如果采用普通的hash函数时，当存储数据的节点数变动，比如扩容，那么就要重新分配数据分布，会导致大量的数据迁移，影响性能
         * 一致性hash的原理是，首先虚拟一个hash环，范围是0-2^32-1(就是一个32位无符号整数范围)，先计算出机器节点对应的hash值，放入到环中对应的位置，然后再根据放入数据的key计算hash值，放入到hash环对应的位置，按照顺时针方向，数据遇到的第一个机器节点就是这个数据应该存放的节点。
         * 根据一致性hash原理，加入扩容增加机器节点，那么只需要移动新增机器节点逆势针方向到上一个节点之间的数据即可。

