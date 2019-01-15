ES上神奇的换行控制

修改logstash上的conf文件中的内容，最主要的是filter中的条件

```
input {
	kafka {
		bootstrap_servers => "kafka地址和端口"
		topics => ["topic名字"]
		consumer_threads => 10
		#codec => "json"
	}
}
 
filter {
	mutate { 
		#这里必须一个手动换行，不能使用\n等符号，否则es不能自动换行
		gsub => [ "message", "\\n", "
	" ] }
}

output {
	elasticsearch {
		user => 用户名
		password => 密码
		pool_max => 最大线程数
		pool_max_per_route => 200
		hosts => ["输出的es地址和端口"]
		index => "es的索引名-%{+YYYY.MM.dd}"
	}
}
```
