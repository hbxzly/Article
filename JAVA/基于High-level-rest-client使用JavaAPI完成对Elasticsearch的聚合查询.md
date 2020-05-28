# 基于High-level-rest-client使用JavaAPI完成对Elasticsearch的聚合查询

@[toc]
**原本是采用 transportClient 来写的，但是一个是官方说明在5.x以后的版本就不怎么支持了，二是因为实际环境上使用了加密，无法通过 transportclient 的方式进行查询了，所以综合了一下采用了 High-level-rest-client 的方式。同时在使用 High-level-rest-client 的方式创建 client 的时候务必注意版本的情况，我这里使用的是5.6版本的，不同版本之间创建 client 的方式的差别还是比较大的。**

**下面进入正题，因为我使用的是 maven 项目的方式，所以首先 pom.xml 里面是我们可能会用上的一些 jar，其次因为该构建客户端的方式需要 java1.8 版本以上，所以我们在 pom.xml 中指定了 java 的版本，并且确认你自己的运行环境中是否成功安装了 java1.8 及以上版本。** 

## pom.xml

```
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.8.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
    </properties>

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

        <!--es for transport-->
        <dependency>
            <groupId>org.elasticsearch</groupId>
            <artifactId>elasticsearch</artifactId>
            <version>5.6.11</version>
        </dependency>
        
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-client</artifactId>
            <version>5.6.11</version>
        </dependency>

	<!--es sniffer-->
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-client-sniffer</artifactId>
            <version>5.6.3</version>
            <scope>compile</scope>
        </dependency>

        <!--es for rest-high-level-client-->
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-high-level-client</artifactId>
            <version>5.6.11</version>
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

        <!--预加载配置信息-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

```


## ElasticsearchConfig
配置使用了lombok的jar包，配置信息保存在 application.properties 中。
```
@Configuration
@Data
public class ElasticsearchConfig {
    @Value("${elasticsearch.host}")
    private String host;
    @Value("${elasticsearch.port}")
    private String port;
    @Value("${elasticsearch.schema}")
    private String schema;
    @Value("${elasticsearch.timeout}")
    private String timeout;
    @Value("${elasticsearch.size}")
    private String size;
    @Value("${elasticsearch.username}")
    private String username;
    @Value("${elasticsearch.password}")
    private String password;
}
```

## ElasticsearchService
在指定的时间范围内（timestamp）查询 关键字对应为 uri 的值为 filtervalue 的信息，timestamp 中的时间范围以时间戳表示
```
@Component
public class ElasticsearchService {
    private static final Logger LOGGER = LoggerFactory.getLogger(ElasticsearchService.class);

    @Autowired
    private ElasticsearchConfig elasticsearchConfig;

    public Boolean getSourceByFilters(String topic, String index, String filtervalue, String timestamp){
        //Host
        String elasticsearchhost = elasticsearchConfig.getHost();
        //Port
        int elasticsearchport = Integer.parseInt(elasticsearchConfig.getPort());
        //Schema
        String elasticsearchschema = elasticsearchConfig.getSchema();
        //Timeout (seconds)
        int elasticsearchtimeout = Integer.parseInt(elasticsearchConfig.getTimeout());
        //Single request quantity
        int elasticsearchsize = Integer.parseInt(elasticsearchConfig.getSize());
        //From what time
        String gte = timestamp.split(",")[0];
        //To what time
        String lte = timestamp.split(",")[1];

        //login es username
        String elasticsearchusername = elasticsearchConfig.getUsername();
        //login es password
        String elasticsearchpassword = elasticsearchConfig.getPassword();

        //topicname prefix
        String fulltopicname = kafkaConfig.getTopicPrefix() + topic;

        try {
            final CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
            credentialsProvider.setCredentials(AuthScope.ANY,
                    //es账号密码
                    new UsernamePasswordCredentials(elasticsearchusername, elasticsearchpassword));
            //自动扫描网段，使用该服务可以在配置信息中仅配置一个es地址即可
            SniffOnFailureListener sniffOnFailureListener = new SniffOnFailureListener();
            //Low Level Client init
            RestClient lowLevelRestClient = RestClient.builder(
                    new HttpHost(elasticsearchhost, elasticsearchport, elasticsearchschema)
            ).setHttpClientConfigCallback(new RestClientBuilder.HttpClientConfigCallback() {
                        @Override
                        public HttpAsyncClientBuilder customizeHttpClient(HttpAsyncClientBuilder httpAsyncClientBuilder) {
                            return httpAsyncClientBuilder.setDefaultCredentialsProvider(credentialsProvider);
                        }
                    })
                    //监听同网段服务
                    .setFailureListener(sniffOnFailureListener)
                    .build();
            SnifferBuilder snifferBuilder = Sniffer.builder(restClient).setSniffIntervalMillis(elasticsearchConfig.getProducerSnifferinterval());
             if (elasticsearchConfig.getProducerFailuredelay() > 0) {
                        snifferBuilder.setSniffAfterFailureDelayMillis(elasticsearchConfig.getProducerFailuredelay());
             }
            sniffOnFailureListener.setSniffer(snifferBuilder.build());
            //High Level Client init
            RestHighLevelClient client = new RestHighLevelClient(lowLevelRestClient);

            final Scroll scroll = new Scroll(TimeValue.timeValueMinutes(1L));
            SearchRequest searchRequest = new SearchRequest(index);
            searchRequest.scroll(scroll);
            SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();

            //Aggregate statement
            searchSourceBuilder.query(QueryBuilders.boolQuery()
                    //.must(QueryBuilders.queryStringQuery(filters))
                    //.must(QueryBuilders.termsQuery("uri",filters))
                    .must(QueryBuilders.matchPhraseQuery("uri",filtervalue))
                    .must(QueryBuilders.rangeQuery("@timestamp").gte(gte).lte(lte))
                    )
                    .timeout(new TimeValue(elasticsearchtimeout,TimeUnit.SECONDS))
                    .size(elasticsearchsize);

            searchRequest.source(searchSourceBuilder);

            //Print the executed DSL statement, which can be used directly in kibana
            //LOGGER.info(searchSourceBuilder.toString());

            SearchResponse searchResponse = client.search(searchRequest);
            if (searchResponse.getHits().totalHits == 0){
                return null;
            }else {
                Boolean kafkaresstatus = new Boolean(true);
                if ("OK".equals(searchResponse.status().toString())){
                    List list = new ArrayList<>();

                    String scrollId = searchResponse.getScrollId();
                    SearchHit[] searchHits = searchResponse.getHits().getHits();
                    for (SearchHit hit : searchResponse.getHits().getHits()) {
                        String res = hit.getSourceAsString();
                    }

                    long totalHits = searchResponse.getHits().getTotalHits();
                    long length = searchResponse.getHits().getHits().length;
                    LOGGER.info("A total of [{}] data was retrieved, and the number of data processed [{}]",totalHits, length);

                    while (searchHits != null && searchHits.length > 0) {

                        SearchScrollRequest scrollRequest = new SearchScrollRequest(scrollId);
                        scrollRequest.scroll(scroll);
                        searchResponse = client.searchScroll(scrollRequest);

                        for (SearchHit hit : searchResponse.getHits().getHits()) {
                            String res = hit.getSourceAsString();
                        }
                        length += searchResponse.getHits().getHits().length;
                        LOGGER.info("A total of [{}] data was retrieved, and the number of data processed [{}]",totalHits, length);
                        if (length == totalHits){
                            break;
                        }
                    }
                    ClearScrollRequest clearScrollRequest = new ClearScrollRequest();
                    clearScrollRequest.addScrollId(scrollId);
                    ClearScrollResponse clearScrollResponse = client.clearScroll(clearScrollRequest);
                    boolean succeeded = clearScrollResponse.isSucceeded();
                    return kafkaresstatus;
                }else {
                    return false;
                }
            }

        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

}
```
