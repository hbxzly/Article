@[TOC](Linux常用命令)

## Linux
1.查看当前路径下指定所有包含00099的文件的总大小，单位kb 
```
du -sk 00099 | awk '{c+=$1}END{print c}'
```

2.查看指定路径下包含00099的文件的总大小

```
find /data/test/day -name "00099" |xargs du -ck
```

3.查看指定路径下指定类型文件的包含指定内容的总行数
```
find 00099.dat.l | xargs grep "2019-04-07 20" | wc -l
```

4.查看当前进程号的所有tcp链接
```
lsof -p 11771 -nP | grep TCP
```

## Vim
1.vim打开文件后删除指定长度的内容（修改里面的10）
```
:%s/^.{10}//
```

2.批量删除指定列
```
:1 切换到行首
ctrl+v  这样会启动可视模式，按 j/k 可以发现它能够在一列上面选中字符
按下 G 这样可以从文本的第一行选中到最后一行
按下 x 就会把这一列删掉
 :sort 排序
```

3.修改文件格式从DOS到UNIX
```
vim dos.txt
:set fileformat=unix
:w
```

## Curl
1.上传文件 
```
[root]# curl http://127.0.0.1:18081/file/batchUpload -X POST -F "filePath=@/data/shell/1554947856915.log"  -F "filePath=@/data/shell/1554947856915.log" --header "Content-Type:multipart/form-data"  -v
```
```
[root]# curl http://127.0.0.1:18081/file/batchUpload -X POST -F "filePath=@/data/shell/1554947856915.log"  -F "filePath=@/data/shell/1554947856915.log" --header "Content-Type:multipart/form-data"  -v
Note: Unnecessary use of -X or --request, POST is already inferred.
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to 127.0.0.1 (127.0.0.1) port 18081 (#0)
> POST /file/batchUpload HTTP/1.1
> Host: 127.0.0.1:18081
> User-Agent: curl/7.57.0
> Accept: */*
> Content-Length: 963
> Content-Type: multipart/form-data; boundary=------------------------d426837656228ad7
> 
< HTTP/1.1 200 OK
< Server: gunicorn/19.9.0
< Date: Tue, 23 Apr 2019 07:24:28 GMT
< Connection: close
< Content-Type: text/html; charset=utf-8
< Content-Length: 178
< 
* Closing connection 0
{"code": "000000", "desc": "success", "data": {"1554947856915.log": "000000", "1554947856915.log": "000000"}}
```

## Kafka
1. 查看指定topic的偏移量
```
bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list 127.0.0.1:9092 --topic test-log --time -1
```




