# Java获取指定topic每个分区的当前偏移量
## 首先引入pom.xml
```xml
<dependencies>
	<dependency>
		<groupId>org.springframework.kafka</groupId>
		<artifactId>spring-kafka</artifactId>
		<!--<version>2.1.10.RELEASE</version>-->
	</dependency>
</dependencies>
```

## 配置properties.properties文件
> 因为我们需要获取的是每个消费者消费的topic的每个分区的当前偏移量，所以在properties配置文件中只需要配置消费者即可
> 不同的group_id消费相同的topic，当前的偏移量也会不一样的

```sql
#kafka消费者配置
spring.kafka.consumer.bootstrap-servers=
spring.kafka.consumer.auto-offset-reset=earliest
spring.kafka.consumer.enable-auto-commit= false
spring.kafka.consumer.fetch-max-wait=30s
spring.kafka.consumer.fetch-min-size=
spring.kafka.consumer.group-id=
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.max-poll-records=500
```

## 执行获取topic每个分区的当前偏移量代码
```java
package cn.migu.log.service.impl;


import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.TopicPartition;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.kafka.KafkaProperties;
import org.springframework.kafka.config.KafkaListenerEndpointRegistry;
import org.springframework.kafka.listener.MessageListenerContainer;
import org.springframework.stereotype.Service;

import java.util.*;

/*
 * @version 1.0 created by LXW on 2019/11/14 14:52
 */
@Service
public class KafkaService {

    private static final Logger LOGGER = LoggerFactory.getLogger(KafkaService.class);

    private final KafkaProperties properties;

    @Autowired
    private KafkaListenerEndpointRegistry registry;

    public KafkaService(KafkaProperties properties) {
        this.properties = properties;
    }

    /**
     * 获取当前topic下的全部分区的偏移量信息
     *
     * @param partitions Collection<TopicPartition> partitions
     * @return {partition:offset}
     */
    public Map<TopicPartition, Long> getTopicPartitionsOffset(Collection<TopicPartition> partitions) {
        KafkaConsumer consumer = null;
        try {
            consumer = new KafkaConsumer(properties.buildConsumerProperties());
            Map<TopicPartition, Long> endOffsets = consumer.endOffsets(partitions);
            return endOffsets;
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }finally {
            consumer.close();
        }
    }

    /**
     * 获取当前服务消费的topic的每个分区的当前偏移量
     * @return {
     *          topic:
     *           {
     *             partitionInfo:offset
     *           }
     *         }
     */
    public Map<String, Map<TopicPartition, Long>> getTopicPartitionsOffset(Set<String> topics){
        List<TopicPartition> partitionsList = new ArrayList<TopicPartition>();
        Map<TopicPartition, String> topicPartitionMap = new HashMap<>();
        Map<String, Map<TopicPartition, Long>> mapMap = new HashMap<>();
        try {
            for (MessageListenerContainer messageListenerContainer : registry.getListenerContainers()) {
                for (TopicPartition topicPartition : messageListenerContainer.getAssignedPartitions()) {
                    topicPartitionMap.put(topicPartition, topicPartition.topic());
                }
            }
            for (String topic : topics) {
                for (TopicPartition topicPartition : topicPartitionMap.keySet()) {
                    if (topic.equals(topicPartitionMap.get(topicPartition))){
                        partitionsList.add(topicPartition);
                    }
                }
                Map<TopicPartition, Long> topicPartitionsOffset = getTopicPartitionsOffset(partitionsList);
                mapMap.put(topic, topicPartitionsOffset);
            }
            return mapMap;
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }
}
```
