---
title: flink
date: 2021-07-13 23:46:50
categories:
tags:
---
## flink
1. flink执行图：
   * streamGraph-->jobGraph-->executionGraph

2. flink的两种状态
   * operator state
   * keyed state
3. flink状态一致性级别
   * 精确一次
   * 至少一次
   * 做多一次
4. 几种状态后端
   * MemoryStateBackend
   * FsStateBackend
   * RocksDBStateBackend
5. flink的常用数据类型
   * java的基础数据类型
   * tuple
   * pojo
   * ArrayList,HashMap
6. flink中的时间类型
   * event time
   * processing time
   * ingestion time

7. flink on yarn
   * 两种模式：
      * session模式: 所有任务共享一个flink 集群，任务与任务之间会有影响，不用
      * pre-job模式: 每个任务一个独立的flinK集群,但是jar包的解析，生成JobGraph都是在客户端，当用户多时，客户端压力大
      * application模式：每个任务一个独立的flinlk集群，
   * Flink On Yarn依赖Yarn资源调度和管理，动态申请资源启动Flink Cluster。每个被Yarn调度运行的Flink Cluster即为一个Application。Application包含了完成的Flink角色JobManager（ApplicationMaster/Container）、TaskManager（Container) 

# flink笔记

## 1.checkpoint -->snapshot
周期性的自动做状态快照，记录任务所有节点再某一时刻的处理状态，便于任务异常重启时从异常时间点进行状态恢复，仿佛没有发生任何异常。
执行checkpinting时，会在流中插入一个barrier，相当于一个屏障，用来区分某一个节点的前与后；

## 2.savepoint
手动创建一个保存点，用于停止任务然后重启，从某一个指定的时间点进行重新运行

## 3.event-time and watermark
event-time 描述真实的事件发生时间
使用event-time就需要提供timestamp提取器和watermark生成器
watermark的作用:This is precisely what watermarks do — they define when to stop waiting for earlier events.
带时间戳的事件流在不确定是否有序的情况下进行传输，使用watermark来衡量何时停止等待早期事件，在时间窗口中经常使用。

event-time处理依赖watermark生成器，根据timestamp生成一个watermark(特殊时间戳)，然后插入到事件流中。
假设一个watermark=t,则代表可以认为时间T<=t的事件流都已经全部到达了。当后期一个T < t 的事件流再次出现时，可以认为此事件流为迟到的事件。

对于迟到的事件，默认是删除。也可以通过side-output方式输出到其他流中进行针对处理。

## 4.window 功能可以用keyedProcessFunction来间接实现

## 5.exactly-once
任务异常时，消息的消费就会存在丢失或者重复。那么flink的消息处理会面临以下三种：
1. Flink makes no effort to recover from failures (at most once)
2. Nothing is lost, but you may experience duplicated results (at least once)
3. Nothing is lost or duplicated (exactly once)

at most once意味着没有开启checkpointing功能;
exactly-once的开启意味着barrier对齐功能开启；
at least once的开启意味着barrier对齐功能没有开启；

barrier对齐指的是，一个operator假设接收两个以上的输入流，当发生一次checkpoint时，每个输入流都会插入一个barrier，并且这些barrier都会流到这个operator上，如果开始对齐功能，则这个operator上的checkpoint会等待所有的barrier收集齐了才能继续进行后续数据的传输，先到的barrier需要等待后到的barrier并且缓存自己数据线源源不断的流数据；如果不开启对齐功能，则每个数据流自己barrier处理完后，operator会继续处理它后面的新数据，不需要关注其他输入流是否完成了barrier,此时当发生故障重启时，某些先checkpoint的数据流的数据就会被重新消费，因为一次完整的state需要所有的operator都完成checkpoint；

## 6.Exactly Once End-to-end
端到端的一致性提供更好到的
To achieve exactly once end-to-end, so that every event from the sources affects the sinks exactly once, the following must be true:
 1. your sources must be replayable, and
 2. your sinks must be transactional (or idempotent)

## 7.keyed stream & keyed state
通过keyBy处理后的流，假设下游需要保存一个keyed state，那个每个key都会有相应的逻辑分区，实际上每个key的数据都是独立的，并不会多个key的state保存在同一个state中，不会常规的都放在同一个map的结构，key/value中的state value会被严格的绑定到event's key。

## 8.checkpoint,event time,watermark,exactly-once关系
1. checkpoint是用来给数据流进行状态快照的，有没有开启checkpoint意味着任务重启时是否能保留之前的处理状态，这是一个单独的参数
2. event-time与watermark必须绑定，这个是用来定义消息的实际时间和判断是否继续等待早期事件的依据时间点,这两个参数与checkpoint是否开启是无关的。
3. exactly-once语句和checkpoint绑定的，依赖状态快照，但是与watermark是无关的；

## 9.table api and sql api
flink的流批一体是通过table 和sql功能实现的，目前table planner推荐使用Blink planner;
flink sql的环境设置分三步：
EnvironmentSettings settings = EnvironmentSettings.newInstance().useBlinkPlanner().inStreamingMode().build();
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tEnv = StreamTableEnvironment.create(env, settings);

flink sql里面的execute()方法不用显示调用了。

## 10 flink sql 控制并发度

## 11.flink keyby
被keyby分流的后的数据会进入一个逻辑分区，可以理解为同样的一个key会被路由到同样的分区，但是一个分区可以接受不同的key划分过来的数据流，flink是通过散列的方式进行逻辑划分的，所以在keyby后面的计算单元中，需要注意一点，如果创建个容器进行装数据，那么如果是非state方式比如使用常规的hashmap进行存放数据，
那么这个hashmap实例数和并发数相等，同一个hashmap实例会被分配给hash(key)相同的不同key共享使用。而如果使用状态进行存储例如mapstate，那么因为state是根据key进行唯一绑定的，
所以不同的key就会有自己唯一对应的state进行存储数据。


## 12.flink taskmanager 内存
* 从1.10后，flink对内存分配进行了升级。其中需要注意managed memory的使用；
* managed memory 分配在堆外非直接内存中，其作用是：Batch jobs用来排序、存放HashTable、中间结果的缓存；Streaming jobs的RocksDB State Backend Memory。因此在streaming jobs中并且没有使用rocksdb statebackend时，这个值应该调整的比较小，可以为0，苏宁调整为flink memory的0.04，flink默认为0.4。
  
## 13.flink taskmanager slot cpu memory
* flink任务都是运行在taskmanager中，jobmanager主要负责任务的调度等，taskmanager 的内存可以人为的自定义大小。
* taskmanager中的slot默认为1个，可以设置多个，taskmanager中的cpu个数默认和slot是1:1比例的，每个slot会的内存是根据taskmanager进行平均分配的，内存资源是隔离的，但是cpu不是。
* 每个taskmanager只有一个jvm进行，slot相当于线程，多个slot是复用一个jvm的连接等资源。
* 实际实践中，根据taskmanager的大小，可以自行决定采用一个slot还是多个slot的方式来部署计算单元。
* flink的可以根据需要执行jar包是人为的设置一些参数的大小，例如manager memory通过-yD taskmanager.memory.managed.fraction=0.04进行设置，slot数量通过-ys 2 进行设置。

## 14.flink 中每个slot与线程的关系
