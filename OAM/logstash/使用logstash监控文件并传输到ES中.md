因为现在有一个需求是不再使用logstash监听kafka发送数据到es，而是监听文件然后发送数据到es。并且，我们文件是txt的文件，文件中的每一行数据是一个标准的json。因此，我们可以使用下面的config进行配置。

```
input {
    file {
        type => "monitor"
        path => ["/data/monitor/filelog/*.txt"]
        codec => json
     }
}

filter {
    mutate
        {
            gsub => [ "message", "\\n", "
" ]
        }
}

output {
    elasticsearch
        {
            user => admin
            password => admin
            pool_max => 1000
            pool_max_per_route => 200
            hosts => ["127.0.0.1:9200"]
            index => "monitor-%{+YYYY.MM.dd}"
        }
}
```
