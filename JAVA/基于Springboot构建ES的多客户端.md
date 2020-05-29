> 有时候我们操作es的时候会有一些特殊的需求，例如需要操作的index使用了不同的es服务器、用户名、密码、参数等，这个时候我们需要使用不同的es的客户端进行操作，但是我们又不希望拆分成多个项目进行使用，这个时候我们就需要在我们的配置中自己构建一套ES的多客户端了。**

## pom.xml
> 首先是我们的pom.xml：

```java
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
        <version>2.0.5.RELEASE</version>
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

    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-lang3</artifactId>
        <version>3.4</version>
    </dependency>

    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.39</version>
    </dependency>

    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-core</artifactId>
        <version>2.9.1</version>
    </dependency>

    <!-- SpringBoot 热启动 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
    </dependency>

    <dependency>
        <groupId>org.apache.commons</groupId>
        <artifactId>commons-pool2</artifactId>
        <version>2.4.2</version>
    </dependency>

    <dependency>
        <groupId>org.elasticsearch.client</groupId>
        <artifactId>elasticsearch-rest-client-sniffer</artifactId>
        <version>5.6.0</version>
    </dependency>

    <dependency>
        <groupId>commons-io</groupId>
        <artifactId>commons-io</artifactId>
        <version>2.6</version>
    </dependency>
</dependencies>
```

## ElasticsearchConfig.java
> 然后是我们的配置文件，我这里使用的是application.properties的配置文件，因为我们使用不同的信息，所以这里我就不写了，可以根据需求自行获取。

## ElasticsearchRestClient.java
```java
import cnkj.site.config.ElasticsearchConfig;
import lombok.extern.slf4j.Slf4j;
import org.apache.http.HttpHost;
import org.apache.http.auth.AuthScope;
import org.apache.http.auth.UsernamePasswordCredentials;
import org.apache.http.client.CredentialsProvider;
import org.apache.http.client.config.RequestConfig;
import org.apache.http.impl.client.BasicCredentialsProvider;
import org.elasticsearch.client.RestClient;
import org.elasticsearch.client.RestClientBuilder;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.client.sniff.SniffOnFailureListener;
import org.elasticsearch.client.sniff.Sniffer;
import org.elasticsearch.client.sniff.SnifferBuilder;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/*
 * @version 1.0 created by LXW on 2018/11/22 9:43
 */
@Slf4j
@Configuration
public class ElasticsearchClient {

    @Bean(name = "HighESClient")
    public RestClient restTomcatClient(ElasticsearchConfig elasticsearchConfig) {
        final CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
        credentialsProvider.setCredentials(AuthScope.ANY,
                //es账号密码
                new UsernamePasswordCredentials(elasticsearchConfig.getUsername(), elasticsearchConfig.getPassword()));
        //自动扫描网段
        //监听同网段服务
        //Low Level Client init
        RestClientBuilder builder = RestClient.builder(
                new HttpHost(
                        elasticsearchConfig.getHost(),
                        Integer.valueOf(elasticsearchConfig.getPort()),
                        elasticsearchConfig.getSchema()
                )
        ).setHttpClientConfigCallback(
                httpClientBuilder -> httpClientBuilder.setDefaultCredentialsProvider(credentialsProvider)
        ).setRequestConfigCallback(new RestClientBuilder.RequestConfigCallback() {
                    @Override
                    public RequestConfig.Builder customizeRequestConfig(RequestConfig.Builder builder) {
                        builder.setConnectTimeout(elasticsearchConfig.getConnectTimeout());
                        builder.setSocketTimeout(elasticsearchConfig.getSocketTimeout());
                        return builder;
                    }
                })
                .setMaxRetryTimeoutMillis(elasticsearchConfig.getMaxRetryTimeoutMillis());
        builder.setMaxRetryTimeoutMillis(elasticsearchConfig.getMaxRetryTimeoutMillis());
        SniffOnFailureListener sniffOnFailureListener = new SniffOnFailureListener();
        builder.setFailureListener(sniffOnFailureListener);
        RestClient lowLevelRestClient = builder.build();
        SnifferBuilder snifferBuilder = Sniffer.builder(lowLevelRestClient).setSniffIntervalMillis(elasticsearchConfig.getSnifferinterval());
        if (elasticsearchConfig.getFailuredelay() > 0) {
            snifferBuilder.setSniffAfterFailureDelayMillis(elasticsearchConfig.getFailuredelay());
        }
        sniffOnFailureListener.setSniffer(snifferBuilder.build());
        return lowLevelRestClient;
    }
    @Bean(name = "HighLevelESClient")
    public RestHighLevelClient restHighLevelClient(@Qualifier("HighESClient") RestClient restClient) {
        return new RestHighLevelClient(restClient);
    }

}
```

## 最终
> 在需要使用的地方直接通过注入的方式使用不同的客户端

```java
@Resource(name = "HighLevelESClient")
private RestHighLevelClient client;
```
