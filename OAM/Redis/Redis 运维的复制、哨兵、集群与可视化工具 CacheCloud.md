# [Redis运维的复制、哨兵、集群与可视化工具CacheCloud](https://blog.csdn.net/carolcoral/article/details/86480716)

## 1. 基本操作

 1. 查看服务 `info server`
 2. 查看状态 `info replication`
 3. 日志标识 M主节点 S从节点 C子进程 Redis3.0版本以后的日志提供该标识
 4. 客户端信息 `client list` 查看复制相关客户端信息，flags=M为主节点，flags=S为从节点
 5. 设置自动重定向 `redis-cli -p 6379 -c`

## 2. 复制

**命令行**
 1. 主从复制 `slaveof 127.0.0.1 6387`
 2. 断开复制 `slaveof no one`
 3. 重载RDB并保持id不变 `debug reload` 该命令会阻塞当前节点，对于大数据量和无法容忍阻塞环境谨慎使用
 4. 全量复制（不知道id） `psync ？ -1`
 5. 全量复制（知道id） `psync {id} -1`
 6. 部分复制 `psync {id} {offset}`

**配置信息**

 1. 只读模式 `slave-read-only yes` 默认只读
 2. 传输延迟 `repl-disable-tcp-nodelay no`  默认关闭
 3. 设置密码 `requirepass` 默认注释，主节点设置了密码，从节点复制必须配置masterauth参数
 4. 设置超时 `repl-timeout` 大数据量（大于6G）同步时一定注意设置超时，默认超时时间60秒
 5. 缓存溢出 `client-output_buffer_limit slave 256MB 64MB 60` 60秒内连续超过64mb溢出，单次超过256mb溢出，溢出情况主节点自动关闭复制，同步失败
 6. 响应命令 `slave-serve-stale-data` 默认开启，开启后从节点响应全部命令，即使从节点正在执行复制
 7. 请求频率 `repl-ping-slave-period` 控制主节点发送ping命令给从节点的频率
 8. 从节点数量 `min-slaves-to-write`
 9. 从节点延迟性功能 `min-slaves-max-lag`

## 3. 哨兵

**命令行**

 1. 启动sentinel节点 `redis-sentinel redis-sentinel-26379.conf` 或者 `redis-server redis-sentinel-26379.conf --sentinel`
 2. 展示监控的全部主节点信息 `sentinel masters`
 3. 展示监控的指定主节点的信息 `sentinel master <主节点别名>`
 4. 展示指定主节点的全部从节点的信息 `sentinel slaves <主节点别名>`
 5. 展示指定主节点的哨兵节点集合 `sentinel sentinels <主节点别名>`
 6. 查看sentinel相关信息 `redis-cli -h 127.0.0.1 -p 26379 info sentinel`
 7. 动态修改配置 `sentinel set <param> <value>` 支持一下参数设置
 
    |参数|使用方法|
    |:----|:--------|
    |quorum|sentinel set mymaster quorum 2|
    |down-after-milliseconds|sentinel set mymaster down-after-milliseconds 30000|
    |failover-timeout|sentinel set mymaster failover-timeout 360000|
    |parallel-syncs|sentinel set mymaster parallel-syncs 2|
    |notification-script|sentinel set mymaster notification-script /opt/xx.sh|
    |client-reconfig-script|sentinel set mymaster client-reconfig-script /opt/yy.sh|
    |auth-pass|sentinel set mymaster auth-pass masterPassword|
    
    `sentinel set` 命令只对当前sentinel节点有效
    `sentinel set` 命令执行完成后立即生效，无需使用 `config rewrite` 刷新配置
    sentinel 对外不支持 `config` 命令

 8. 返回指定主节点的IP地址和端口 `sentinel get-master-addr-by-name <master-name>`
 9. 当前sentinel节点对符合规则的主节点的配置重置 `sentinel reset <pattern>` 包含清除主节点的相关状态，重新发现从节点和sentinel节点
 10. 对指定主节点执行强制故障转移 `sentinel failover <master-name>` 故障转移完成后其他sentinel节点按照故障转移结果更新自身配置
 11. 检测当前可用sentinel节点总数是否达到设置的 `quorum` 数量 `sentinel ckquorum <master-name>`
 12. 将sentinel节点的配置强制刷到磁盘上 `sentinel flushconfig`
 13. 取消当前sentinel节点对指定主节点的监控 `sentinel remove <master-name>`
 14. 以命令行方式完成sentinel节点对指定主节点的监控 `sentinel monitor <master-name> <ip> <port> <quorum>`
 15. sentinel节点之间交换对主节点是否下线的判断 `sentinel is-master-down-by-addr <ip> <port> <current_epoch> <runid>`
    `ip` 主节点ip
    `port` 主节点端口
    `current_epoch` 当前配置纪元
    `runid` 当为'*'时，sentinel节点直接交换对主节点下线的判定。当为当前sentinel节点的runid时，作用是当前sentinel节点希望目标sentinel节点统一自己成为领导者的请求

 16. 获取sentinel对应主节点的信息 `sentinel get-master-addr-by-name master-name`

**配置信息**

- redis-sentinel-26379.conf
```
port 26379
daemonize yes
logfile "26379.log"
dir /opt/soft/redis/data
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 30000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
```

`sentinel` 节点默认端口 26379
`sentinel monitor <主节点别名> <主节点地址> <主节点端口> <主节点失败后至少需要多少哨兵节点同意才能升级为主节点的数量>`
`port` 代表sentinel节点的端口
`dir` 代表sentinel节点的工作目录
`down-after-milliseconds` sentinel节点连接主节点的超时时间
`parallel-syncs` 从节点对新主节点发起复制操作的从节点个数
`failover-timeout` 故障转移超时时间，作用于故障转移的每个阶段
`sentinel auth-pass <master-name> <password>` 如果主节点设置了密码，哨兵节点同样需要设置相同密码才能访问 
`sentinel notification-script <master-name> <script-path>` 配置故障转移期间通过指定的脚本发送一些告警级别的时间信息，其中script-path是脚本路径 
`sentinel client-reconfig-script <master-name> <script-path>` 故障转移结束后触发指定的脚本

## 4.集群
**命令行**

 1. 查看集群信息 `cluster nodes`
 2. 节点握手 `cluster meet {ip} {port}`
 3. 集群模式下让一个节点成为从节点 `cluster replicate {nodeId}` 该命令必须在对应的从节点上执行
 4. 批量分配槽 `redis-cli -h 127.0.0.1 -p 6379 cluster addslots {0...5461}`
 5. 数据迁移
    5.1 目标节点准备导入槽的数据 `cluster setslot {slot} importing {sourceNodeId}`
    5.2 源节点准备迁出槽的数据 `cluster setslot {slot} migrating {targetNodeId}`
    5.3 源节点循环执行获取count个属于槽{slot}的键 `cluster getkeysinslot {slot} {count}`
    5.4 批量键迁移（3.0.6版本以上提供） `migrate {targetIp} {targetPort} "" 0 {timeout} keys {keys...}`
    5.5 重复执行步骤3和步骤4知道槽下所有的键值数据迁移到目标节点
    5.6 向集群内所有主节点发送命令 `cluster setslot {slot} node {targetNodeId}` 通知槽分配给目标节点

 6. 忘记节点（不再与下线节点通信，不建议线上操作） `cluster forget {downNodeId}`
 7. 查看当前key对应的槽 `cluster keyslot {key}`
 8. 初始化槽和节点映射关系 `cluster slots`
 9. 查询nodeId对应主节点下全部从节点信息 `cluster slaves {nodeId}`
 10. 在指定节点地址上执行 `cluster failover` 完成手动故障迁移
 11. 强制选举 `cluster failover force` 一般用于主节点宕机后无法执行自动故障迁移
 12. 强制迁移 `cluster failover takeover` 一般用于集群内超过一半以上主节点故障场景，慎用

**redis-trib.rb 工具使用**

 1. 创建集群 
    `redis-trib.rb create --replicas 1 127.0.0.1:6481 127.0.0.1:6482 127.0.0.1:6483 127.0.0.1:6484 127.0.0.1:6485 127.0.0.1:6486`
 2. 检查集群完整性 
    `redis-trib.rb check 127.0.0.1:6481`
 3. 为现有集群添加新节点，并直接添加为从节点 
    `redis-trib.rb add-node new_host:new_port existing_host:existing_port --slave --master-id <arg>`
 4. 添加新节点，正式环境建议使用该语句，添加后会自动检测新节点状态
    `redis-trib.rb add-node 127.0.0.1:6385 127.0.0.1:6379`
 5. 槽重分片功能 
    `redis-trib.rb reshard host:port --from <arg> --to <arg> --slots <arg> --yes --timeout <arg> --pipeline <arg>` 
    host:port 必传参数，集群内任意节点地址，用来获取整个集群信息
    --from 制定源节点的id，如果有多个源节点，使用逗号分割，如果是all源节点变为集群内所有主节点，在迁移过程中提示用户输入
    --to 需要迁移的目标节点的id，目标节点只能填写一个，在迁移过程中提示用户输入
    --slots 需要迁移槽的总数量，在迁移过程中提示用户输入
    --yes 当打印出reshard执行计划时，是否需要用户输入yes确认后在执行reshard
    --timeout 控制每次migrate操作的超时时间，默认为60000毫秒
    --pipeline 控制每次批量迁移键的数量，默认是10

 6. 检查节点均衡性 
    `redis-trib.rb rebalance`
 7. 设置节点均衡性 
    `redis-trib.rb rebalance 127.0.0.1:6379`
 8. 忘记节点（下线节点，不再通信） 
    `redis-trib.rb del-node {主节点host:主节点port} {从节点downNodeId}`
 9. 查看指定节点的信息 
    `redis-trib.rb info 127.0.0.1:6379`
 10. 单机数据迁移到集群中
    `redis-trib.rb import host:port --from <arg> --copy --replace`
    迁移只能从单机节点向集群环境导入数据
    不支持在线迁移数据，迁移数据的时候应用方必须停写，无法平滑迁移数据
    迁移过程中途如果出现超时等错误，不支持断点续传只能重新全量导入
    使用单线程进行数据迁移，大数据量迁移速度过慢
    推荐使用唯品会开发的redis-migrate-tool
 
**配置信息**

1. redis-6379.conf
```
# 节点端口
port 6379
# 开启集群模式
cluster-enabled yes
# 节点超时时间，单位毫秒
cluster-node-timeout 15000
# 集群内部配置文件
cluster-config-file "nodes-6379.conf"
```

2. 关闭故障转移集群不可用状态 `cluster-require-full-coverage no`

## 5.Redis监控运维云平台 [CacheCloud](https://github.com/sohutv/cachecloud)

 - 监控统计：	提供了机器、应用、实例下各个维度数据的监控和统计界面。
 - 一键开启：	Redis Standalone、Redis Sentinel、Redis Cluster三种类型的应用，无需手动配置初始化。
 - Failover：	支持哨兵,集群的高可用模式。
 - 伸缩：	提供完善的垂直和水平在线伸缩功能。
 - 完善运维： 提供自动运维和简化运维操作功能，避免纯手工运维出错。
 - 方便的客户端 方便快捷的客户端接入。
 - 元数据管理： 提供机器、应用、实例、用户信息管理。
 - 流程化： 提供申请，运维，伸缩，修改等完善的处理流程
 - 一键导入： 一键导入已经存在Redis

