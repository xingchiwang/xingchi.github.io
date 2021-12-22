## HDFS的设计
HDFS以流式数据访问模式来存储超大文件，运行于商用硬件集群上。<br>
·超大文件<br>
·流式数据访问<br>
·商用硬件<br>
·低时间延迟的数据访问<br>
·大量的小文件<br>
·多用户写入，任意修改文件<br>‘

## 使用场景
·超大文件的存储<br>
·一写多读（高效的访问模式）。HDFS目前只支持单个写入者，它使用append的方式写数据，不支持在文件的任意位置修改数据。<br>
·高吞吐量要求。HDFS不适合低时延数据访问的场景（因为写操作节点间会有多次RPC通信）使用，它是为高数据吞吐量应用而优化，但它是以提高时延为代价的。<br>
·批处理场景。Hadoop每次计算都涉及HDFS上数据集的大部分或者是全部数据，但HDFS是流式数据访问，可以把文件当做数据流输入和输出，不能随机访问。

## 数据块
HDFS同样也有块的概念（block）的概念，但是大得多，默认为128MB。<br>
HDFS的块比磁盘的块大，其目的是为了最小化寻址开销。如果块足够大，从磁盘传输数据的时间会明显大于定位这个块开始位置所需的时间。<br>
MapReduce中的map任务通常一次只处理一个块中的数据。<br>

## 对数据块进行抽象的好处
1.一个文件的大小可以大于网络中任意一个磁盘的容量。<br>
2.使用抽象快而非整个文件作为存储单元，大大简化了村塾子系统的设计。<br>
3.块还非常适合用于数据备份进而提供数据容错能力和提高可用性。将每个块复制到少数几个物理上相互独立的机器上，可以确保在块、磁盘或机器发生故障后数据不会丢失。

## namenode和datanode
HDFS集群有两类节点以管理节点-工作节点模式运行，即一个namenode(管理节点)和多个datanode（工作节点）。<br>
1.namenode<br>
namenode管理文件系统的命名空间。它维护这文件系统树及整棵树内所有的文件和目录。这些信息以两个文件形式永久的保存在本地磁盘上：命名空间镜像文件（fsimage）和编辑日志文件（edits_*）。<br>
它也会维护hdfs每个文件各个块所在的datanode信息（文件与数据块的映射关系），但该位置信息不会永久保存，而是存放在内存中，重启系统后消失。<br>
2.datanode<br>
datanode是文件系统的工作节点。它们根据需要存储并检索数据块，定期向namenode发送它们所存储的块的列表。<br>
当datanode不能正常工作时，HDFS会自动启动该datanode上所有数据的复制任务，将丢失的数据重新分布到其他datanode上。<br>
3.secondary namenode<br>
HDFS的辅助节点，它被用来创建检查点，定期合并edits_*和fsimage，以防止edits_*文件过大。<br>
一般可以将它放在另一台单独的物理机上运行，因为它需要占用大量的CPU时间，并且需要与namenode一样大的内存来执行合并操作，但它保存的状态总是滞后于namenode，当namenode失效时，难免会丢失部分数据，所以HDFS本身不能通过secondary namenode实现高可用。<br>


## HDFS高可用
导致HDFS无法提供正常服务的原因：NameNode升级和维护，软硬件维护和软件故障（硬件故障不是主要原因），错误操作。<br>
雅虎的数据表明:在雅虎运行的 15 个集群中，三年时间内，只有 3 次 NameNode 的故障与硬件问题有关。<br>
通过联合使用namenode和secondary namenode可以在一定程度上避免数据丢失，但是依然无法实现文件系统的高可用。<br>
如果没有高可用，那么想要从一个失效的namenode恢复，系统管理员得启动一个拥有文件系统元数据复本的namenode,并配置datanode和客户端以便使用这个新的namenode。新的namenode直到满足以下情形才能响应服务：<br>
1、将fsimage载入内存；<br>
2、重演编辑日志edits_；<br>
3、接收到足够多的来自datanode的数据块报告并推出安全模式。对于一个大型并拥有大量文件和数据块的集群，namenode冷启动需要30分钟甚至更久。<br>
<br>
现有比较成熟的HA解决方案有：<br>
1、元数据备份<br>
利用 Hadoop 自身的 Failover 措施(通过配置 dfs.name.dir)，NameNode 可以将元数据信息保存到多个目录。<br>
冷备，恢复时间与文件系统规模成正比。<br>
2、Secondary NameNode方案<br>
冷备，恢复时间与文件系统规模成正比，恢复的元数据并不是 HDFS 此刻的最新数据，存在一致性问题<br>
3、Checkpoint Node方案<br>
利用Hadoop的Checkpoint机制，自建Checkpoint节点进行备份，同SecondaryNameNode。<br>
4、Facebook的Avatarnode方案<br>
HDP2.0增加了对高可用的支持，它通过一对活动-备用namenode来实现，即Active NameNode 和 Standby NameNode互备。<br>
HDFS实际上是借助zookeeper实现高可用<br>



## namenode容错机制
1.备份那些组成文件系统元数据持久状态的文件。
2.另一种可行的方法是运行一个辅助namenode，但它不能被用作namenode。

## 复本怎么放
namenode如何选择在那个datanode存储复本？这里需要对可靠性、写入带宽和读取带宽进行权衡。例如，吧所有复本都存储在一个节点损失的写入带宽最小，同一机架上服务器间的读取带宽是很高的。

## 网络拓扑与Hadoop
在海量数据处理中，其主要限制因素是节点之间数据的传输速率-带宽很稀缺。
![image](https://user-images.githubusercontent.com/44181286/142754568-0f8abf3b-adca-4140-9ae3-26b4eb240b04.png)

• distance(/dl/rl/nl, /dl/rl/nl) = 0 (processes on the same node)

• distance//'dl/rl/nl, /dl/rl/n2) = 2 (different nodes on the same rack)

• distance(/dl/rl/nl, Zdl/r2/n3) = 4 (nodes on different racks in the same data center)

• distance(/dl/rl/nl, Zd2/r3/n4) = 6 (nodes in different data centers)

## 文件读过程
客户端将要读取的文件路径发送给namenode，namenode获取文件的元信息（主要是block的存放位置信息）返回给客户端，客户端根据返回的信息找到相应datanode逐个获取文件的block并在客户端本地进行数据追加合并从而获得整个文件

![image](https://user-images.githubusercontent.com/44181286/147022115-e2f083e3-2166-489c-bdac-d378dc49a621.png)

读详细步骤：

1、跟namenode通信查询元数据（block所在的datanode节点），找到文件块所在的datanode服务器

2、挑选一台datanode（就近原则，然后随机）服务器，请求建立socket流

3、datanode开始发送数据（从磁盘里面读取数据放入流，以packet为单位来做校验）

4、客户端以packet为单位接收，先在本地缓存，然后写入目标文件，后面的block块就相当于是append到前面的block块最后合成最终需要的文件。

## 文件写过程
![image](https://user-images.githubusercontent.com/44181286/147022411-819a3c36-5322-4dac-a2a7-aca62a797669.png)

写详细步骤：

1、根namenode通信请求上传文件，namenode检查目标文件是否已存在，父目录是否存在

2、namenode返回是否可以上传

3、client会先对文件进行切分，比如一个blok块128m，文件有300m就会被切分成3个块，一个128M、一个128M、一个44M请求第一个 block该传输到哪些datanode服务器上

4、namenode返回datanode的服务器

5、client请求一台datanode上传数据（本质上是一个RPC调用，建立pipeline），第一个datanode收到请求会继续调用第二个datanode，然后第二个调用第三个datanode，将整个pipeline建立完成，逐级返回客户端

6、client开始往A上传第一个block（先从磁盘读取数据放到一个本地内存缓存），以packet为单位（一个packet为64kb），当然在写入的时候datanode会进行数据校验，它并不是通过一个packet进行一次校验而是以chunk为单位进行校验（512byte），第一台datanode收到一个packet就会传给第二台，第二台传给第三台；第一台每传一个packet会放入一个应答队列等待应答

7、当一个block传输完成之后，client再次请求namenode上传第二个block的服务器。


如果任何datanode在数据写入期间发生故障。首先关闭管线，确认吧队列中的所有数据包都添加回数据队列的最前端，以确保故障节点下游的datanode不会漏掉任何一个数据包。为存储在另一正常datanode的当前数据块制定一个新的表示，并将改表示传给namenode，以便故障datanode在恢复后可以删除存储的部分数据块。



