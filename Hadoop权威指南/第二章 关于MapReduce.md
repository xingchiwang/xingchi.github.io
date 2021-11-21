## 数据流
MapReduce作业（job）是客户端需要执行的一个工作单元：它包括输入数据、MapReduce成语和配置信息。Hadoop将作业分成若干个任务（task）来执行，其中包括两类任务：map任务和reduce任务。这些任务运行在集群的节点上，并通过YARN进行调度。

