## HDFS的设计
HDFS以流式数据访问模式来存储超大文件，运行于商用硬件集群上。<br>
·超大文件<br>
·流式数据访问<br>
·商用硬件<br>
·地时间延迟的数据访问<br>
·大量的小文件<br>
·多用户写入，任意修改文件<br>

## 数据块
HDFS同样也有块的概念（block）的概念，但是大得多，默认为128MB。<br>
HDFS的快比磁盘的块大，其目的是为了最小化寻址开销。如果块足够大，从磁盘传输数据的时间会明显大于定位这个块开始位置所需的时间。<br>
MapReduce中的map任务通常一次只处理一个块中的数据。<br>

## 对数据块进行抽象的好处
1.一个文件的大小可以大于网络中任意一个磁盘的容量。<br>
2.使用抽象快而非整个文件作为存储单元，大大简化了村塾子系统的设计。<br>
3.块还非常适合用于数据备份进而提供数据容错能力和提高可用性。将每个块复制到少数几个物理上相互独立的机器上，可以确保在块、磁盘或机器发生故障后数据不会丢失。

## namenode和datanode
HDFS集群有两类节点以管理节点-工作节点模式运行，即一个namenode(管理节点)和多个datanode（工作节点）。namenode管理文件系统的命名空间。它维护这文件系统树及整棵树内所有的文件和目录。这些信息以两个文件形式永久的保存在本地磁盘上：命名空间镜像文件和编辑日志文件。<br>
客户端代表用户通过namenode和datanode交互来访问整个文件系统。<br>
datanode是文件系统的工作节点。它们根据需要存储并检索数据块，冰洁定期想namenode发送它们所存储的块的列表。<br>

## namenode容错机制
1.备份那些组成文件系统元数据持久状态的文件。
2.另一种可行的方法是运行一个辅助namenode，但它不能呗用作namenode。

## 网络拓扑与Hadoop
在海量数据处理中，其主要限制因素是节点之间数据的传输速率-带宽很稀缺。
![image](https://user-images.githubusercontent.com/44181286/142754568-0f8abf3b-adca-4140-9ae3-26b4eb240b04.png)

• distance(/dl/rl/nl, /dl/rl/nl) = 0 (processes on the same node)

• distance//'dl/rl/nl, /dl/rl/n2) = 2 (different nodes on the same rack)

• distance(/dl/rl/nl, Zdl/r2/n3) = 4 (nodes on different racks in the same data center)

• distance(/dl/rl/nl, Zd2/r3/n4) = 6 (nodes in different data centers)


