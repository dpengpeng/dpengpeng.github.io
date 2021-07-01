---
title: kafka自问自答
date: 2021-07-01 22:25:22
categories: kafka
tags: kafka
---
## kafka是什么
kafka是一开始是个消息中间件引擎，后来有了kafka streams后便可以提供流计算功能(以jar形式添加到一个应用中即可使用)
<!-- more -->
## kafka的consumer应知应会
* kafka的consumer分为standalone consumer 和 consumer group
* standalone consumer 运行时只有一个消费者实例，单线程消费
* consumer group 以组为单位进行消费topic,每个partition都会分配一个consumer，每个consumer可以有一个或者多个partition
* kafka内部有个消费者位移主题__consumer_offset,默认50个partition，每个consumer group首先都会注册到50个partition中的某一个partition中，并且这个partition的leader所在的broker就是这个consumer group的协调者Coordinator,所有的consumer在加入组之前都要寻找到他所在组的Coordinator，然后等所有的consumer都加入组以后，Coordinator就会让leader consumer(第一个找到coordiantor的consumer)分配消费方案，然后coordinator就会将分配方案分发给各个consumer，这样整个consumer group就开始进行消费信息了
* consumer group 会发生rebalance，此时整个consumer group就会停止消费，重新通过coordinator来分配新的消费方案才行
* consumer group发生rebalance情景包括：新增consumer、consumer实例进程挂掉、consumer与coordinator的心跳响应超时、订阅的topic个数发生变化、topic的partitoin个数增加

## kafka的producer应知应会
* producer没有group的概念，每个producer 实例就一个独立的消费生产者
* producer 0.11版本后提供了幂等性和事务性，为发送消息提供原子性保障，也即精确一次语义。幂等性只能保证单次会话单个分区的原子性，单次会话的某个分区不会出现重复消息，producer重启就不生效了。事务性则可以保证多个分区且重启之后仍然可以有原子性，事务性producer是read committed隔离级别的，所有的消息要么都成功，要么都失败。
* producer 发送有个ack机制，发送后的消息符合ack规则才算成功提交，kafka只对已提交的消息提供一致性保障，并且根据ack的不同，保障性界别也不同。ack=all为最高级别，所有的partition副本都确认了才算提交成功。

## kafka broker应知应会
* kafka broker 是通过zookeeper来进行分布式协调管理的，zookeeper中的watch机制可以让客户端是实时监控到znode节点的变化并作出相应的动作

## topic && partition
* 每个topic都会一个或者多个partition，每个partition都有一个或者多个副本
* partition中分leader和follower角色，follower专门负责同步leader的消息，不读外提供读写服务
* leader 会维护一个isr列表，里面放的是与leader同步的follwer，包括leader自己。虽然follwer不一定能实时与leader保持同步，不过只要符合一定规则就算同步(默认是follower延迟不超过leader 10s)
* partition 里面有不同的游标：HW，LEO，offset。HW为高水位，为多个partition共识的可以被consumer消费的位移临界值(不包含hw处)，leo为一个partition最后一条消息的offset+1，代表放下一条消息的offset
* 当leader partition挂了以后，其他的follower就会竞选leader，当某个follower变成leader之后其他的follower就会来同步他的消息，如果某个follower的leo大于leader的leo，那么多余的消息就会被截断，因为这些消息不算已提交的，所以producer检测到上一次发送失败后就会重新发送消息，然后也会自动过滤掉重复的消息，进而将消息重新完整的发送到新的leader上面。