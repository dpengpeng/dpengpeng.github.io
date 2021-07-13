---
title: clickhouse
date: 2021-07-13 23:46:10
categories:
tags:
---
# clickhouse集群搭建
clickhouse是一种olap型数据库，采用列式存储，适合进行分析类业务，sql语句丰富且与mysql语句类似。

## 采用的集群架构
vip --> chproxy集群 --> clickhouse集群(3*2)

## vip
vip的一种架构方式：keepalived+haproxy，分别部署到两台机器
haproxy负责分发请求，keepalived负责维护haproxy节点的failover和vip的竞选。

## chproxy集群
* [官网地址](https://github.com/Vertamedia/chproxy)
* chproxy集群每个节点单独部署，都是独立的节点，通过vip来负载均衡访问每个chproxy，任何一节点挂了以后不影响集群服务，只需要拉起来就行了。
* Chproxy集群模式,是ClickHouse数据库的http代理和负载均衡器，读写请求都可以通过chproxy配置策略来访问ClickHouse集群。
* write列表配置数据直接写入本地表,read列表配置数读取分布式表

## clickhouse集群
### 前期准备
* [官网地址](https://clickhouse.yandex)
* 搭建ch集群需要zookeeper
* ClickHouse集群模式，采用3*2模式，3个分片shard，每个shard 2个副本，两个副本互相同步数据，同时对外接收读写请求;
* zookeeper集群用来存储ClickHouse集群的节点信息和分布式表等信息，起到协调ClickHouse集群作用;
  
## 参考链接
* https://blog.csdn.net/qq_37933018/article/details/111462868
* https://www.jianshu.com/p/762f8b7d323d
* https://zhuanlan.zhihu.com/p/103781296
* https://cloud.tencent.com/developer/article/1488512


1.clickhouse私用相关文章
运维派clickhouse在mysql上的应用
http://www.yunweipai.com/archives/28786.html
https://www.secrss.com/articles/3173

2.解决clickhouse并发问题之CHproxy安装配置
https://wchch.github.io/2019/06/14/%E8%A7%A3%E5%86%B3clickhouse%E5%B9%B6%E5%8F%91%E9%97%AE%E9%A2%98%E4%B9%8BCHproxy%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AE/

3.分布式表建立方式
https://www.altinity.com/blog/2017/6/5/clickhouse-data-distribution
https://www.altinity.com/blog/2018/5/10/circular-replication-cluster-topology-in-clickhouse

集群搭建参考
http://www.clickhouse.com.cn/topic/5a366e97828d76d75ab5d5a0
https://www.cnblogs.com/freeweb/p/9352947.html
https://www.jianshu.com/p/5f7809b1965e
https://blog.csdn.net/hyesc/article/details/83022236
https://www.jianshu.com/p/ae45e0aa2b52
https://www.jianshu.com/p/20639fdfdc99
https://blog.csdn.net/Alice_qixin/article/details/89209519

### 访问ch
1. 官方JDBC方式直连clickhouse sever:
```java
   <dependency>
      <groupId>ru.yandex.clickhouse</groupId>
      <artifactId>clickhouse-jdbc</artifactId>
      <version>0.2</version>
   </dependency>
```
集群模式下可以采用负载均衡方式直连，配置后端服务检查，可以自动过滤故障节点，使用方式如下：
```java
public static BalancedClickhouseDataSource buildBalancedClickhouseDataSource() {
     ClickHouseProperties properties = new ClickHouseProperties();
     //直连clickhouse server
     properties.setUser("use");
     properties.setPassword("pass");
     BalancedClickhouseDataSource dataSource = new BalancedClickhouseDataSource    ("jdbc:clickhouse://ip1:8123,ip2:8123,ip3:8123/databasename", properties);
     //连接检查机制必须打开,否则服务端任意一个宕机后整个集群不可用
     dataSource.scheduleActualization(10, TimeUnit.SECONDS);
     return dataSource;
}
```
2. 通过http连接池方式，自定义http client客户端工具，连接chproxy集群，发送select 和 insert 请求.现有的方式是使用一个线程池，通过线程池来发送http任务。

### 集群安装
1. 官网yum源下载rpm包,官网有下载方式
2. 第三方打包好的安装包:https://packagecloud.io/Altinity/clickhouse

使用集群安装工具
使用clustershell安装:clush -g all -b 'rpm -ivh /home/dpp/clickhouse-server-19.11.9.52-1.el7.x86_64.rpm'

安装时需要关闭swap
swap关闭命令:
1.查看swap空间和使用情况
swapon -s
free -m
2.关闭swap
swapoff /dev/dm-1
swapon -s
3.启用之前关闭分区
swapon /dev/dm-1
swapon -s

包名及安装顺序如下：
* rpm -ivh clickhouse-server-common-19.11.9.52-1.el7.x86_64.rpm
* rpm -ivh clickhouse-common-static-19.11.9.52-1.el7.x86_64.rpm
* rpm -ivh clickhouse-server-19.11.9.52-1.el7.x86_64.rpm
* rpm -ivh clickhouse-debuginfo-19.11.9.52-1.el7.x86_64.rpm
* rpm -ivh clickhouse-client-19.11.9.52-1.el7.x86_64.rpm

安装前需要下载依赖包：
libtool-ltdl-2.4.2-21.el7_2.x86_64.rpm
unixODBC-2.3.1-11.el7.x86_64.rpm
安装异常参考：https://clickhouse.yandex/docs/en/operations/troubleshooting/

3. 卸载及删除安装文件
* yum list installed | grep clickhouse
* yum remove -y clickhouse-common-static.xxx
* yum remove -y clickhouse-server-common.xxx
  
* rm -rf /var/lib/clickhouse
* rm -rf /etc/clickhouse-*
* rm -rf /var/log/clickhouse-server

### ClickHouse 表引擎
1. ReplacingMergeTree:  
* 此为MergeTree家族里具有复制功能的表引擎，需要配合zookeeper使用；采用此引擎的表，同一个shard里面的不同副本的本地表会根据zookeeper里面的表节点信息，互相增量同步彼此没有的分区数据。
* insert数据时，会随机挑选某个shard里面的某一个副本节点进行插入操作，其他副本后台进行同步数据，达到不同副本之间的数据一致性和可靠性。
* select数据通过distributed table 读取数据，http请求被分配到某一个分布式表的节点上，此节点再将sql分发到其他分片，并行执行sql语句，然后将数据汇聚到初始http请求节点，返回给用户。图片解析可以百度: clickhouse reading from distributed table
  
### 建表语句
建表需要创建两种表:本地表和分布式表
本地表示例：
```sql
CREATE TABLE IF NOT EXISTS dbname.tablename ON CLUSTER `cluster-name`
(
`db_id` Int32, 
`status_name` String, 
`status_value` String, 
`time_stamp` DateTime DEFAULT now()
)ENGINE = ReplicatedMergeTree('/clickhouse/clustername/dbname/tablename/{shard}', '{replica}')
PARTITION BY toYYYYMMDD(time_stamp)
ORDER BY (db_id, time_stamp, status_name);
```
'/clickhouse/clustername/dbname/tablename/{shard}': The path to the table in ZooKeeper.
'{replica}'：The replica name in ZooKeeper.
两个变量的值均从配置文件中自动读取替换。

分布式表：
```sql
CREATE TABLE dbname.tablename_all ON CLUSTER `clustername` AS dbname.tablename
ENGINE = Distributed('clustername', 'dbname', 'local_tablename', rand());
```
note: MergeTree家族引擎必须要有一个 Date 的列来作为索引。

### 常见问题
1. 表的建法
* 本地表和分布式表都采用集群模式sql语句建立，不要在每台机器上一个一个建，防止元数据同步不一致

2. sql写法的小建议
* join时大表在左小表在右,JOIN操作时一定要把数据量小的表放在右边，ClickHouse中无论是Left Join 、Right Join还是Inner Join永远都是拿着右表中的每一条记录到左表中查找该记录是否存在，所以右表必须是小表。
* 多表关联时，需在子查询里增加过滤条件，尽量提前缩小数据范围。
* 如果不想加where条件，那么可以提前构建大宽表或者预计算
* clickhouse的in用法比join效率高；多个列可以组合成一个tuple类型使用，可以结合in使用
* 筛选分区select ... from xxx where partition_expr=xxx

3. 副本的failover方式
* 一个shard里面的不同副本是可以同时对外服务的，正常情况下每个副本可以根据配置，接收insert和select 类sql，如果某个副本不可用，chproxy会自动重发到其他可用副本。

4. clickhouse索引模式
* clickhouse采用稀疏索引，默认粒度8192，每隔固定长度会在索引相关文件里设置索引信息，涉及数据文件cloum_name.bin和标记文件cloum_name.mrk2，clickhouse中的数据是每个列一个文件，内部采用压缩模式存储。适合检索大数据量下的数据集，server会根据sql中的索引字段自动检索到适合的范围内数据块，然后会读取索引区间之间的所有数据。稀疏索引会引起额外的数据读取，当读取主键单个区间范围的数据时，每个数据块中最多会多读 index_granularity * 2 行额外的数据。大部分情况下，当 index_granularity = 8192 时，ClickHouse的性能并不会降级。

5. 集群方式删除表分区
* ALTER TABLE table_name ON CLUSTER 'cluster_name' DROP PARTITION partition_expr
举例：alter table mon_database_status on cluster 'cluster-1' drop partition 20191104
其中，partition_expr表达式不可用单引号/双引号修饰，否则会报异常 : DB::Exception: There was an error on [snk207:9000]: Cannot execute replicated DDL query on leader.
这个方式可以只需要在某个server节点上执行一次就可以删除集群中所有的此表的词分区，不用每个分片一个一个执行。

6. clickhouse的复制表中的leader说明
* Leader replica just coordinates some background processes, leveraging ZooKeeper cluster. So unlike master/slave setup in other DBMS, in ClickHouse you shouldn't care about replica leadership status for reads and writes.
More details are over here: https://clickhouse.yandex/docs/en/operations/table_engines/replication/

7. zookeeper对replicate table的影响
如果clickhouse服务启动正常，此时zookeeper不可用，那么clickhouse中的replicate table会变成read_only模式；
同时，在insert的过程中，如果zookeeper不可用，或者clickhouse与zookeeper交互有异常，都会抛出异常，执行失败；

8. 修改不同分片的权重比例
* 场景1：通过chproxy来插入数据，则在配置文件中，为负载均衡的服务节点设定不同分片的比例，比如将<204,205>设置为1/3,<206,207>设置为2/3，如下所示，实测生效;
nodes:
[
"snk204:8123",
"snk205:8123",
"snk206:8123",
"snk207:8123",
"snk206:8123",
"snk207:8123"
]

目前测试，修改完配置文件后重启生效.
chproxy可以修改配置文件后动态加载，不用重启服务，方法为：ps -ef|grep -v grep |grep -v nohup|grep chproxy|awk '{print $2}'|xargs kill -HUP，此为想进程发送SIGHUP信号,让进程重新加载服务。
* 场景2：clickhouse的配置文件中的分片权重
通过在<shard>标签里面的属性设置<weight>1</weight>，这种情况是通过分布式表插入数据时，分配的比重。

9. clickhoues的TTL功能bug
目前使用的版本为:ClickHouse version 19.11.9.52. 对此版本的表级别TTL进行试验测试，TTL无效，官方证实为bug，说是在11.14版本修复。

10. 物化视图(materialized view)
物化视图的作用：
1.时序层面物化：从一个表衍生划分出不同的时间维度表的表
2.维度层面物化：针对不同的维度衍生出不同维度的不同engine表
物化视图的建法：
1.分片多副本模式下，用on cluster集群方式建立replicated*模式表，在每个节点上创建一个本地物化视图，再在集群上创建一个分布式表来查集群的物化视图;
2.其他模式下自选

11. truncate 删除表数据默认禁止超过50G的数据，需要按提示进行操作，创建一个force文件后，就可以删除了。
12. order by是对表中的数据进行排序存放,主键可以和排序排序键不一样,但是必须是排序键的前缀,也就是order by的前面几个字段


### 集群扩容
1. 新增分片
即给集群新增若干分片，比如从2个分片，增加到3个分片  
> 处理步骤：
>> 1.准备好新增分片的机器  
>> 2.在新增机器上安装好clickhouse各个元件，修改一份最新全集群的配置文件，然后同步到集群所有节点，根据分片号，修改相应标签值。clickhouse的配置文件是动态生效的。此时，已经可以查询到新增分片了  
>> 3.启动新分片的机器，此时zookeeper中没有新分片信息，然后在新节点上创建一份与其他节点一致的库和表（此时的建库建表应该在每个节点使用本地模式执行，不能加 on cluster语句，{shard},{replicate}的占位符依旧可以使用），此时zookeeper已经存在新分片的表信息了，并且新节点可以查询分布式表中的数据和执行集群相关操作  
>> 4.此时的新分片还未对外服务，不能接受插入数据请求。因此在chproxy中添加节点信息，重启chproxy服务，新分片即可对外服务，同时给新增分片增加数据权重，达到数据均衡。等到数据大致平衡后再修改一次分片权重。
方案实测生效。

2. 新增副本
>> 1.副本的Clickhouse安装同上
>> 2.更改新的添加副本的配置文件，同步到所有节点
>> 3.启动新的副本节点，创建库和表，此时新的副本就会自动同步数据了，并且未在chproxy中登记，暂时可以不对外服务
>> 4.将新的副本增加到chproxy的均衡节点中，就可以正常使用了。
方案实测生效。
副本的减少操作类似

### 集群维护
1. chproxy集群中某台服务宕机，则直接拉起服务继续使用
2. 一个shard中的某个副本节点宕机，则
  * 重新拉起CK服务，如果启动正常，则会自动同步丢失的数据
  * 若副本节点所在机器损坏不可用，则直接更换部分机器，并更新节点中的配置文件，用新的副本替换原来的副本，新增副本的方法见《新增副本》,zookeeper中会自动删除下线副本节点。
3. 扩容分片方法，见《新增分片》方法;剔除分片zookeeper是不会自动删除节点的。

### 数据搬迁
当遇到数据需要从一个集群迁移到另一个集群时，有如下方法：
1. 方法一：
通过remote表函数来实现，这种方式主要是用来每次执行一个表的数据迁移,表的数据量不是太大，语句格式为：
insert into db.table select * from remote('addresses_expr', db.table[, 'user'[, 'password']]);
在需要导入数据的新集群执行。

2. 方法二：
通过clickhouse-copier工具来迁移，这种方式可以处理任意大小的集群数据，可以通过配置文件来同时以此同步不同的表数据，需要与zookeeper结合使用；

命令为：
clickhouse-copier copier --config /etc/clickhouse-sever/zookeeper.xml --task-path /clickhouse/copytasks/task1 --task-file schema.xml  --base-dir /data/clickhouse-copier --task-upload-force true 

(官方--daemon方式运行不起作用，有待考究)
或者自己实现后台执行模式:
clickhouse-copier copier --config /etc/clickhouse-sever/zookeeper.xml --task-path /clickhouse/copytasks/task1 --task-file /etc/clickhouse-sever/schema.xml  --base-dir /data/clickhouse-copier --task-upload-force true >/dev/null 2>&1 &
其中：--config,--task-path,--base-dir涉及到的文件或者目录可以自定义位置

参数含义：
Parameters:
daemon — Starts clickhouse-copier in daemon mode.
config — The path to the zookeeper.xml file with the parameters for the connection to ZooKeeper.
task-path — The path to the ZooKeeper node. This node is used for syncing clickhouse-copier processes and storing tasks. Tasks are stored in $task-path/description.
task-file — Optional path to file with task configuration for initial upload to ZooKeeper.
task-upload-force — Force upload task-file even if node already exists.
base-dir — The path to logs and auxiliary files. When it starts, clickhouse-copier creates clickhouse-copier_YYYYMMHHSS_<PID> subdirectories in $base-dir. If this parameter is omitted, the directories are created in the directory where clickhouse-copier was launched.

命令会把task-file指定的文件上传到zookeeper节点路劲task-file下，无需人工上传任务文件.
实验测试使用步骤如下：
1.准备zookeeper.xml配置文件
```xml 
<yandex>
<logger>
<level>trace</level>
<size>100M</size>
<count>3</count>
</logger>
<zookeeper>
<node index="1">
<host>snk204</host>
<port>2181</port>
</node>
</zookeeper>
</yandex>
```
2.准备任务描述文件schema.xml，里面把数据的流转方式填写完善
3..执行迁移数据命令
clickhouse-copier copier --config /etc/clickhouse-sever/zookeeper.xml --task-path /clickhouse/copytasks/task1 --task-file /etc/clickhouse-sever/schema.xml  --base-dir /data/clickhouse-copier --task-upload-force true >/dev/null 2>&1 &
或者
clickhouse-copier copier --config /etc/clickhouse-sever/zookeeper.xml --task-path /clickhouse/copytasks/task1 --task-file /etc/clickhouse-sever/schema.xml  --base-dir /data/clickhouse-copier --task-upload-force true
4.查看任务同步状态
在/data/clickhouse-copier目录下会创建一个当前任务的子目录，可以通过查看下面的日志来了解任务同步状态.
等待数据同步完成即可。
NOTE:
clickhouse-copier每个同步完的分区会写在zookeeper的节点中，当重复执行命令时，并不会对已经完成的分区数据重复执行。并且clickhouse-copier一旦读取完数据就会停止任务进程。
因此这种方式适合于同步那些分区数据不会再变化的数据。

3. 方法三：
通过修改clickhouse的集群配置文件，给每个分片增加新副本但是新副本可以不对外服务的方式，将老副本的数据同步到新副本，待数据同步差不多时，剔除旧的副本和机器，再次修改集群配置文件，将上游任务重定向到新的集群中即可。

### clickhosue 中一些常用命令
1. clickhouse-client -u default -h 127.0.0.1 -m
2. clickhouse-client -u default -h 127.0.0.1 -m -q "xxx_sql"
3. systemctl stop clickhouse-server.service
4. systemctl restart clickhouse-server.service
5. systemctl status clickhouse-server.service
6. sudo service clickhouse-server status
7. alter table db.table drop partition 20210408

### 额外补充
1. mysql语法中的执行顺序:
SELECT语句的完整语法为：
(7) SELECT
(8) DISTINCT <select_list>
(1) FROM <left_table>
(3) <join_type> JOIN <right_table>
(2) ON <join_condition>
(4) WHERE <where_condition>
(5) GROUP BY <group_by_list>
(6) HAVING <having_condition>
(9) ORDER BY <order_by_condition>
(10) LIMIT <limit_number>

案例:
 SELECT a.customer_id, COUNT(b.order_id) as total_orders
 FROM table1 AS a
 LEFT JOIN table2 AS b
 ON a.customer_id = b.customer_id
 WHERE a.city = 'hangzhou'
 GROUP BY a.customer_id
 HAVING count(b.order_id) < 2
 ORDER BY total_orders DESC;

2. group by语句的写法需要注意最后结果的唯一性,不能使select里面的字段产生某个列有多个值,并且select里面的字段要么在group by里面,要么用聚合函数来计算
clickhouse中的语法:
!!!clickhouse语法中写明order by顺序在select之前,和mysql中的顺序不一样.
如果存在GROUP BY子句，则在该子句中必须包含一个表达式列表。其中每个表达式将会被称之为“key”.
SELECT，HAVING，ORDER BY子句中的表达式列表必须来自于这些“key”或聚合函数。
简而言之，被选择的列中不能包含非聚合函数或key之外的其他列。
与MySQL不同的是（实际上这是符合SQL标准的），你不能够获得一个不在key中的非聚合函数列（除了常量表达式）。
但是你可以使用‘any’（返回遇到的第一个值）、max、min等聚合函数使它工作。

3. 当通过proxy写ck时,proxy到CK服务端大量的tcp_tw来源于分片中副本之间的复制造成
当前端写proxy的过程中,如果http端为短连接,那么就会在proxy端产生大量的tcp_tw,如果为长链接,那么在proxy端会复用tcp连接.
但是http端的请求不会影响CK服务端,只有chproxy端才会影响CK.


