# Topic基本操作

### 在linux服务器上查找kafka的安装位置

>locate kafka-topics.sh

* kafka配置文件在kafka的安装文件夹下的 ./config/service.properties 中

* partitions指定topic分区数，replication-factor指定topic每个分区的副本数

	* partitions分区数:
	
		* partitions ：分区数，控制topic将分片成多少个log。可以显示指定，如果不指定则会使用broker(server.properties)中的num.partitions配置的数量
		* 虽然增加分区数可以提供kafka集群的吞吐量、但是过多的分区数或者或是单台服务器上的分区数过多，会增加不可用及延迟的风险。因为多的分区数，意味着需要打开更多的文件句柄、增加点到点的延时、增加客户端的内存消耗。
		* 分区数也限制了consumer的并行度，即限制了并行consumer消息的线程数不能大于分区数
		* 分区数也限制了producer发送消息是指定的分区。如创建topic时分区设置为1，producer发送消息时通过自定义的分区方法指定分区为2或以上的数都会出错的；这种情况可以通过alter –partitions 来增加分区数。
	* replication-factor副本
		* replication factor 控制消息保存在几个broker(服务器)上，一般情况下等于broker的个数。
		* 如果没有在创建时显示指定或通过API向一个不存在的topic生产消息时会使用broker(server.properties)中的default.replication.factor配置的数量

### 创建kafka topic

>bin/kafka-topics.sh --create --topic topicname --replication-factor 1 --partitions 1 --zookeeper localhost:2181

1.topicname 是你要创建的topic的名称

2.--replication-factor 指定partition的replicas数，建议设置为2；

 3.--partitions 指定分区数，这个参数需要根据broker数和数据量决定，正常情况下，每个broker上两个partition最好； 
 
### 查看kafka topic

>bin/kafka-topics.sh --zookeeper localhost:2181 --list

* 其中zookeeper后面的 localhost:2181 是zookeeper的地址和端口

### 向kafka topic 中生产数据（生产数据）

>bin/kafka-console-producer.sh --broker-list localhost:9092 --topic t_cdr

### 向kafka topic 中消费数据（查看数据）

>bin/kafka-console-consumer.sh  --zookeeper node01:2181  --topic t_cdr --from-beginning

### 查看topic某分区偏移量最大（小）值

>bin/kafka-run-class.sh kafka.tools.GetOffsetShell --topic hive-mdatabase-hostsltable  --time -1 --broker-list node86:9092 --partitions 0

* time为-1时表示最大值，time为-2时表示最小值

### 增加topic分区数

* 为topic t_cdr 增加10个分区

>bin/kafka-topics.sh --zookeeper node01:2181  --alter --topic t_cdr --partitions 10

### 删除topic，慎用，只会删除zookeeper中的元数据，消息文件须手动删除

>bin/kafka-run-class.sh kafka.admin.DeleteTopicCommand --zookeeper node01:2181 --topic t_cdr

### 查看topic消费进度

* 这个会显示出consumer group的offset情况， 必须参数为--group， 不指定--topic，默认为所有topic

>Displays the: Consumer Group, Topic, Partitions, Offset, logSize, Lag, Owner for the specified set of Topics and Consumer Group

<pre>

		bin/kafka-run-class.sh kafka.tools.ConsumerOffsetChecker
	
		required argument: [group] 
		Option Description 
		------ ----------- 
		--broker-info Print broker info 
		--group Consumer group. 
		--help Print this message. 
		--topic Comma-separated list of consumer 
			  topics (all topics if absent). 
		--zkconnect ZooKeeper connect string. (default: localhost:2181)
			
		Example,
			
		bin/kafka-run-class.sh kafka.tools.ConsumerOffsetChecker --group pv
			
		Group           Topic              Pid Offset   logSize    Lag    Owner 
		pv              page_visits        0   21       21         0      none 
		pv              page_visits        1   19       19         0      none 
		pv              page_visits        2   20       20         0      none
</pre>
