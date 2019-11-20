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

```java
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
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.PartitionInfo;
import org.apache.kafka.common.TopicPartition;

import java.util.*;

/*
 * @version 1.0 created by LXW on 2019/11/20 10:20
 */
public class KafkaUtil {


    /**
     * 获取当前topic下的全部分区的偏移量信息
     *
     * @param properties 配置信息
     * @param partitions Collection<TopicPartition> partitions
     * @return {partition:offset}
     */
    public static Map<TopicPartition, Long> getPartitionsOffset(Map<String, Object> properties, Collection<TopicPartition>
            partitions) {
        KafkaConsumer consumer = new KafkaConsumer(properties);
        try {
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
     *
     * @param properties 配置信息
     * @param topics Collection<String> topics
     * @return {
     *          topic:
     *           {
     *             partitionInfo:offset
     *           }
     *         }
     */
    public static Map<String, Map<TopicPartition, Long>> getTopicPartitionsOffset(Map<String, Object> properties, Set<String> topics){
        Map<String, Map<TopicPartition, Long>> topicPartitionMap = new HashMap<>();
        KafkaConsumer kafkaConsumer = new KafkaConsumer(properties);
        try {
            for (String topic : topics) {
                List<PartitionInfo> partitionsInfo = kafkaConsumer.partitionsFor(topic);
                Set<TopicPartition> topicPartitions = new HashSet<>();
                for (PartitionInfo partitionInfo : partitionsInfo) {
                    TopicPartition topicPartition = new TopicPartition(partitionInfo.topic(), partitionInfo.partition());
                    topicPartitions.add(topicPartition);
                }
                Map<TopicPartition, Long> topicPartitionsOffset = getPartitionsOffset(properties, topicPartitions);
                topicPartitionMap.put(topic, topicPartitionsOffset);
            }
            return topicPartitionMap;
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }finally {
            kafkaConsumer.close();
        }
    }

}
```

## How to use
```java
    public static void main(String[] args) {
        Set topics = new HashSet(Arrays.asList("test1", "test2"));
        KafkaProperties kafkaProperties = new KafkaProperties();
        kafkaProperties.setBootstrapServers(Arrays.asList("127.0.0.1:9200"));
        kafkaProperties.setClientId("clientId");
        Map<String, Object> consumerProperties = kafkaProperties.buildConsumerProperties();
        Map<String, Map<TopicPartition, Long>> serviceTopicPartitionsOffset = KafkaUtil.getTopicPartitionsOffset(consumerProperties, topics);
        // TODO waht you want to do
    }
```
其中properties可以直接通过properties文件自动注入的方式自动加载进去
