## 一、MySQL主从同步机制
MySQL主从同步机制主要分为以下四个步骤：
1.主库对所有DDL和DML产生的日志写进Binlog;
2.主库生成一个Log dump线程，用来给从库I/O线程读取Binlog;
3.从库的I/O Thread去请求主库的Binlog,并将得到的Binlog日志写到Relay log文件夹中；
4.从库的SQL Thread会读取Relay log文件中的日志并解析成具体操作，将主库的DDL和DML操作事件在从库中重放

## 二、主从同步延迟怎么产生的？
在MySQL5.7之前，主从复制都是单线程的操作，主库对所有的DDL和DML产生的日志写进binlog，由于binlog是顺序写，所以效率很高。Slave的SQL Thread线程将主库的DDL和DML操作事件在slave中重放。DML和DDL的IO操作是随机的，不是顺序的，成本高很多。

另一方面，由于SQL Thread也是单线程的，当主库的并发较高时，产生的DML数量超过了slave的SQL Thread所能处理的速度，或者当slave中有大型query语句产生了锁等待那么延时就产生了。

产生主从延迟常见原因包括主库**负载过高、网络延迟、机器性能太低、配置不合理**等。


[Link](url) and ![Image](src)
