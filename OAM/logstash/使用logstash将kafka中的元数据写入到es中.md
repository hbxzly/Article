## 目前用的是logstash7.3版本，匹配kafka2.0以上。[Logstash7.3 kafka input doc](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-kafka.html)

## logstash7.3目前支持的元数据只有，header元数据目前logstash暂时不支持

> 通过源代码可以看出 [logstash-input-kafka](https://github.com/logstash-plugins/logstash-input-kafka/blob/master/lib/logstash/inputs/kafka.rb)

```ruby
if @decorate_events
	event.set("[@metadata][kafka][topic]", record.topic)
	event.set("[@metadata][kafka][consumer_group]", @group_id)
	event.set("[@metadata][kafka][partition]", record.partition)
	event.set("[@metadata][kafka][offset]", record.offset)
	event.set("[@metadata][kafka][key]", record.key)
	event.set("[@metadata][kafka][timestamp]", record.timestamp)
```


* [metadata][kafka][topic]: Original Kafka topic from where the message was consumed.
* [@metadata][kafka][consumer_group]: Consumer group
* [@metadata][kafka][partition]: Partition info for this message.
* [@metadata][kafka][offset]: Original record offset for this message.
* [@metadata][kafka][key]: Record key, if any.
* [@metadata][kafka][timestamp]: Timestamp in the Record. Depending on your broker 

## 配置文件如下：

* 重点在于filter中的 [mutate](https://www.elastic.co/guide/en/logstash/current/plugins-filters-mutate.html) 属性的使用

```java
input {
        kafka {
            bootstrap_servers => "127.0.0.1:9092"
            topics => ["test"]
            group_id => "test"
            #如果使用元数据就不能使用下面的byte字节序列化，否则会报错
            #key_deserializer_class => "org.apache.kafka.common.serialization.ByteArrayDeserializer"
            #value_deserializer_class => "org.apache.kafka.common.serialization.ByteArrayDeserializer"
            consumer_threads => 1
            #默认为false，只有为true的时候才会获取到元数据
			decorate_events => true
			auto_offset_reset => "earliest"
         }
}
filter {
	mutate {
		#从kafka的key中获取数据并按照逗号切割
		split => ["[@metadata][kafka][key]", ","]
		add_field => {
			#将切割后的第一位数据放入自定义的“index”字段中
			"index" => "%{[@metadata][kafka][key][0]}"
		}
	}
}
output {
   elasticsearch {
          user => elastic
          password => changeme
          pool_max => 1000
          pool_max_per_route => 200
          hosts => ["127.0.0.1:9200"]
          index => "test-%{+YYYY.MM.dd}"
   }
    stdout {
        codec => rubydebug
    }
}

```
