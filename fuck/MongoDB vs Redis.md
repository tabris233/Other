MongoDB和Redis都是NoSQL，采用结构型数据存储。二者在使用场景中，存在一定的区别，这也主要由于二者在内存映射的处理过程，持久化的处理方法不同。

 MongoDB建议集群部署，更多的考虑到集群方案，Redis更偏重于进程顺序写入，虽然支持集群，也仅限于主-从模式。

 

比较指标	|MongoDB(v2.4.9)	|Redis(v2.4.17)	|比较说明
-|-|-|-
实现语言	|c++	|c/c++	|-
协议	|BSON,自定义二进制	|类telnet	|-
性能	|依赖内存,TPS{(transaction per second)代表每秒执行的事务数量}较高	|依赖内存,TPS非常高	|Redis优于MongoDB
可操作性	|丰富的数据表达,索引;最类似于关系型数据库,支持丰富的查询语句	|数据丰富,较少的IO	|MongoDB优于Redis
内存及存储	|适合大数据量存储,依赖系统虚拟内存,采用镜像文件存储;内存占用率比较高,官方建议独立部署在64位系统	|Redis2.0后支持虚拟内存特性(VM) 突破物理内存限制;数据可以设置时效性,类似于memcache	|不同的应用场景,各有千秋
可用性|	支持master-slave,replicatset(内部采用paxos选举算法,自动故障恢复),auto sharding机制,对客户端屏蔽了故障转移和切片机制	|依赖客户端来实现分布式读写;主从复制时,每次从节点重新连接主节点都要依赖整个快照,无增量复制;不支持auto sharding,需要依赖程序设定一致性hash机制	|MongoDB优于Redis；单点问题上,MongoDB应用简单,相对用户透明,Redis比较复杂,需要客户端主动解决.(MongoDB一般使用replicasets和sharding相结合,replicasets侧重高可用性以及高可靠,sharding侧重性能,水平扩展)
可靠性|	从1.8版本后,采用binlog方式(类似Mysql) 支持持久化	|依赖快照进行持久化;AOF增强可靠性;增强性的同时,影响访问性能	|-
一致性|	不支持事务,靠客户端保证	|支持事务,比较脆,仅能保证事务中的操作按顺序执行	|Redis优于MongoDB
数据分析|	内置数据分析功能(mapreduce)	|不支持	|MongoDB优于Redis
应用场景|	海量数据的访问效率提升	|较小数据量的性能和运算	|MongoDB优于Redis
