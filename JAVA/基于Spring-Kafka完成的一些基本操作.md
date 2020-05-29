# 基于Spring-Kafka 完成的一些基本操作

> 原本写 kafka-client 的时候使用的是原生的kafka连接 org.apache.kafka，但是发现不好对缓存容量以及对握手次数的控制，所以后面换用了 spring-kafka ，下面的代码是基于 spring-kafka 完成的开发。功能包含：查询指定的 topic 并判断是否存在、 以指定的分区数和副本数创建 topic、 以及发送数据到指定的 topic 中.

## pom.xml
```java
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka-clients</artifactId>
            <version>1.1.1</version>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
            <version>2.1.9.RELEASE</version>
        </dependency>

        <!-- SpringBoot 热启动 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!--文件操作-->
        <dependency>
            <groupId>commons-io</groupId>
            <artifactId>commons-io</artifactId>
            <version>2.6</version>
        </dependency>

        <!--预加载配置信息-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
```

## KafkaConfig
> 相关配置的功能说明介绍：http://kafka.apachecn.org/documentation.html#producerconfigs

```java
@Configuration
@EnableKafka
@Data
public class KafkaConfig {
    @Value("${spring.kafka.producer.bootstrap.servers}")
    private String hosts;

    @Value("${spring.kafka.producer.key.serializer}")
    private String key;

    @Value("${spring.kafka.producer.value.serializer}")
    private String value;

    @Value("${spring.kafka.producer.acks}")
    private String acks;

    @Value("${spring.kafka.producer.retries}")
    private String retries;

    @Value("${spring.kafka.producer.buffer.memory}")
    private String bufferMemory;

    @Value("${spring.kafka.producer.compression.type}")
    private String compressionType;

    @Value("${spring.kafka.producer.batch.size}")
    private String batchSize;

    @Value("${spring.kafka.producer.client.id}")
    private String clientId;

    @Value("${spring.kafka.connections.max.idle.ms}")
    private String maxConnectionsIdleMs;

    @Value("${spring.kafka.max.request.size}")
    private String maxRequestSize;

    @Value("${spring.kafka.topic.partitions}")
    private String topicPartitions;

    @Value("${spring.kafka.topic.prefix}")
    private String topicPrefix;

    // ----------------producer---------------
    @Bean
    public Map<String, Object> producerConfigs() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, hosts);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.ACKS_CONFIG, acks);
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG,Long.parseLong(bufferMemory));
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG,compressionType);
        props.put(ProducerConfig.RETRIES_CONFIG, retries);
        props.put(ProducerConfig.BATCH_SIZE_CONFIG,Integer.parseInt(batchSize));
        props.put(ProducerConfig.CLIENT_ID_CONFIG,clientId);
        props.put(ProducerConfig.CONNECTIONS_MAX_IDLE_MS_CONFIG, Long.parseLong(maxConnectionsIdleMs));
        props.put(ProducerConfig.MAX_REQUEST_SIZE_CONFIG, Integer.parseInt(maxRequestSize));
        return props;
    }

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        return new DefaultKafkaProducerFactory<>(producerConfigs());
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }

    @Bean
    public KafkaAdmin admin() {
        Map<String, Object> configs = new HashMap<>();
        configs.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, hosts);
        return new KafkaAdmin(configs);
    }
}
```

## KafkaService
```java
public interface KafkaService {

    /**
     * 发送数据到指定的topic中
     *
     * @param topicname topic名称
     * @param data 数据
     * @return 发送的状态
     */
    Boolean sendDataToTopic(String topicname, String data);


    /**
     * 校验topic是否已经存在于kafka中
     *
     * @param topicname topic的名称
     * @return 是否存在的状态
     */
    Boolean isExistTopic(String topicname);


    /**
     * 创建指定的topic
     *
     * @param topicname topic的名称
     * @return 创建topic是否成功的状态
     */
    Boolean createTopic(String topicname);
}
```

## KafkaServiceImpl
```java
@Service
public class KafkaServiceImpl implements KafkaService {

    @Autowired
    private KafkaTemplate kafkaTemplate;

    @Autowired
    private KafkaAdmin kafkaAdmin;

    @Autowired
    private KafkaConfig kafkaConfig;

    @Override
    public Boolean isExistTopic(String topicname) {
        try {
            AdminClient adminClient = AdminClient.create(kafkaAdmin.getConfig());
            ListTopicsOptions listTopicsOptions = new ListTopicsOptions();
            listTopicsOptions.listInternal(true);
            ListTopicsResult res = adminClient.listTopics(listTopicsOptions);
            Boolean flag = res.names().get().contains(topicname);
            adminClient.close();
            return flag;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public Boolean createTopic(String topicname) {
        try {
            Boolean existflag = isExistTopic(topicname);
            Boolean flag = new Boolean(true);
            if (existflag == true){
                flag = true;
            }else {
                AdminClient adminClient = AdminClient.create(kafkaAdmin.getConfig());
                NewTopic newTopic = new NewTopic(topicname,Integer.parseInt(kafkaConfig.getTopicPartitions()),(short)1);
                List<NewTopic> topicList = Arrays.asList(newTopic);
                adminClient.createTopics(topicList);
                adminClient.close();
                flag = isExistTopic(topicname);
            }
            return flag;
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

    @Override
    public Boolean sendDataToTopic(String topicname, String data) {
        ListenableFuture res = kafkaTemplate.send(topicname,data);
        try {
            Boolean flag = new Boolean(true);
            if (res.get() == null){
                flag = false;
            }else if (res.get() != null){
                flag = true;
            }
            return flag;
        } catch (InterruptedException e) {
            e.printStackTrace();
            return false;
        } catch (ExecutionException e) {
            e.printStackTrace();
            return false;
        }
    }
}
```
