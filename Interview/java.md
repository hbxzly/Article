### 以下是近期遇到的一些面试题，欢迎push答案

##### 1. java的8中基本类型
> int short long byte boolean float double char

##### 2. java中的异步机制


##### 3. java中的可重入锁
> `可重入锁`:指的是以线程为单位，当一个线程获取对象锁之后，这个线程可以再次获取本对象上的锁，而其他的线程是不可以的。也就是说同一个线程再次调用lock的时候，可以继续执行lock后面的方法。
> `synchronized` 和 `ReentrantLock` 都是可重入锁。
> 可重入锁的意义之一在于防止死锁。
> 实现原理实现是通过为每个锁关联一个请求计数器和一个占有它的线程。
> 当计数为0时，认为锁是未被占有的；
> 线程请求一个未被占有的锁时，JVM将记录锁的占有者，并且将请求计数器置为1 。
> 如果同一个线程再次请求这个锁，计数器将递增；每次占用线程退出同步块，计数器值将递减。直到计数器为0,锁被释放。
> 关于父类和子类的锁的重入:子类覆写了父类的synchonized方法，然后调用父类中的方法，此时如果没有可重入的锁，那么这段代码将产生死锁

##### 4. 乐观锁和悲观锁的原理
> `悲观锁`:总是假设最坏的情况，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会阻塞直到它拿到锁（共享资源每次只给一个线程使用，其它线程阻塞，用完后再把资源转让给其它线程）。传统的关系型数据库里边就用到了很多这种锁机制，比如行锁，表锁等，读锁，写锁等，都是在做操作之前先上锁。Java中synchronized和ReentrantLock等独占锁就是悲观锁思想的实现。
> `乐观锁`:总是假设最好的情况，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号机制和CAS算法实现。乐观锁适用于多读的应用类型，这样可以提高吞吐量，像数据库提供的类似于write_condition机制，其实都是提供的乐观锁。在Java中java.util.concurrent.atomic包下面的原子变量类就是使用了乐观锁的一种实现方式CAS实现的。

##### 5. 什么情况会导致数据库中索引失效
> 1. 如果条件中有or，即使其中有条件带索引也不会使用(这也是为什么尽量少用or的原因)
> 2. 对于多列索引，不是使用的第一部分(第一个)，则不会使用索引
> 3. like查询是以%开头
> 4. 如果列类型是字符串，那一定要在条件中将数据使用引号引用起来,否则不使用索引
> 5. 如果mysql估计使用全表扫描要比使用索引快,则不使用索引
> 6. 没有查询条件，或者查询条件没有建立索引 
> 7. 在查询条件上没有使用引导列 
> 8. 查询的数量是大表的大部分，应该是30％以上。 
> 9. 索引本身失效
> 10. 查询条件使用函数在索引列上，或者对索引列进行运算(+，-，*，/，! 等) 错误的例子：select * from test where id-1=9; 正确的例子：select * from test where id=10; 
> 11. 对小表查询 
> 12. 提示不使用索引
> 13. 统计数据不真实 
> 14. CBO计算走索引花费过大的情况。其实也包含了上面的情况，这里指的是表占有的block要比索引小。 
> 15. 隐式转换导致索引失效.这一点应当引起重视.也是开发中经常会犯的错误. 由于表的字段tu_mdn定义为varchar2(20),但在查询时把该字段作为number类型以where条件传给Oracle,这样会导致索引失效. 错误的例子：select * from test where tu_mdn=13333333333; 正确的例子：select * from test where tu_mdn='13333333333'; 
> 16. 1,<> 2,单独的>,<,(有时会用到，有时不会) 
> 17. like "%_" 百分号在前. 
> 18. 表没分析. 
> 19. 单独引用复合索引里非第一位置的索引列. 
> 20. 字符型字段为数字时在where条件里不添加引号. 
> 21. 对索引列进行运算.需要建立函数索引. 
> 22. not in ,not exist. 
> 23. 当变量采用的是times变量，而表的字段采用的是date变量时.或相反情况。 
> 24. B-tree索引 is null不会走,is not null会走,位图索引 is null,is not null 都会走 
> 25. 联合索引 is not null 只要在建立的索引列（不分先后）都会走, in null时 必须要和建立索引第一列一起使用,当建立索引第一位置条件是is null 时,其他建立索引的列可以是is null（但必须在所有列 都满足is null的时候）,或者=一个值； 当建立索引的第一位置是=一个值时,其他索引列可以是任何情况（包括is null =一个值）,以上两种情况索引都会走。其他情况不会走。


##### 6. mysql中索引的类型有哪些
> `normal`：表示普通索引
> `unique`：表示唯一的，不允许重复的索引，如果该字段信息保证不会重复例如身份证号用作索引时，可设置为unique
> `full textl`: 表示 全文搜索的索引。 FULLTEXT 用于搜索很长一篇文章的时候，效果最好。用在比较短的文本，如果就一两行字的，普通的 INDEX 也可以。总结，索引的类别由建立索引的字段内容特性来决定，通常normal最常见。

##### 7. es如何实现对数据的快速查询


##### 8. es在底层是以什么方式保存的数据


##### 9. 如何优化es中获取大报文出现的效率问题


##### 10. kafka如何保证了高并发性


##### 11. spring初始化的流程


##### 12. springboot如何初始化javabean


##### 13. spring启动类可以继承哪些接口


##### 14. linux中如何查看内核使用情况


##### 15. linux中如何快速统计文件中指定字符出现的总行数


##### 16. linux中如何打开代码调试的debug模式


##### 17. linux中如何定位内存问题


##### 18. 服务在什么情况会导致cpu的使用率过高


##### 19. hashmap的实现原理


##### 20. 集合类中哪些是有序集合


##### 21. vue.js的生命周期


##### 22. vue.js的特点是什么


##### 23. vue.js适合实现的双向数据绑定


##### 24. String包装类长度是可变的吗


##### 25. ArrayList和LinkedList的区别


##### 26. redis如何实现分布式锁


##### 27. jvm内存调优


##### 28. 创建对象在内存中的实现过程


##### 29. Spring的AOP原理


##### 30. Spring的IOC原理


##### 31. SpringBoot中实现事务处理


##### 32. SpringBoot中的@Transaction的隔离级别有哪些
> `TransactionDefinition.ISOLATION_DEFAULT`：这是默认值，表示使用底层数据库的默认隔离级别。对大部分数据库而言，通常这值就是`TransactionDefinition.ISOLATION_READ_COMMITTED`。
> `TransactionDefinition.ISOLATION_READ_UNCOMMITTED`：该隔离级别表示一个事务可以读取另一个事务修改但还没有提交的数据。该级别不能防止脏读，不可重复读和幻读，因此很少使用该隔离级别。比如PostgreSQL实际上并没有此级别。
> `TransactionDefinition.ISOLATION_READ_COMMITTED`：该隔离级别表示一个事务只能读取另一个事务已经提交的数据。该级别可以防止脏读，这也是大多数情况下的推荐值。
> `TransactionDefinition.ISOLATION_REPEATABLE_READ`：该隔离级别表示一个事务在整个过程中可以多次重复执行某个查询，并且每次返回的记录都相同。该级别可以防止脏读和不可重复读。
> `TransactionDefinition.ISOLATION_SERIALIZABLE`：所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

##### 33. SpringBoot中的@Transaction的传播行为有哪些
> `TransactionDefinition.PROPAGATION_REQUIRED`：如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。这是默认值。
> `TransactionDefinition.PROPAGATION_REQUIRES_NEW`：创建一个新的事务，如果当前存在事务，则把当前事务挂起。
> `TransactionDefinition.PROPAGATION_SUPPORTS`：如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
> `TransactionDefinition.PROPAGATION_NOT_SUPPORTED`：以非事务方式运行，如果当前存在事务，则把当前事务挂起。
> `TransactionDefinition.PROPAGATION_NEVER`：以非事务方式运行，如果当前存在事务，则抛出异常。
> `TransactionDefinition.PROPAGATION_MANDATORY`：如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。
> `TransactionDefinition.PROPAGATION_NESTED`：如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于`TransactionDefinition.PROPAGATION_REQUIRED`。





