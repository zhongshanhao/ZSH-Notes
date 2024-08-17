# hadoop

- 分布式系统架构
- 主要解决海量数据的 存储 和 分析计算 问题
- 泛指Hadoop生态圈

三篇google论文

GFS-->HDFS

Map-Reduce-->MR

BigTable-->HBase

常用端口号

| 端口                      | Hadoop2.x | Hadoop3.x      |
| ------------------------- | --------- | -------------- |
| NameNode内部通信端口      | 8020/9000 | 8020/9000/9820 |
| NameNode HTTP UI          | 50070     | 9870           |
| MapReduce查看执行任务端口 | 8088      | 8088           |
| 历史服务器通信端口        | 19888     | 19888          |

常用配置文件

3.x core-site.xml	hdfs-site.xml	yarn-site.xml	mapred.site.xml	workers

2.x core-site.xml	hdfs-site.xml	yarn-site.xml	mapred.site.xml	slaves

## HDFS

(Hadoop Distributed File System)，分布式文件系统

随着数据量越来越大，一个操作系统存不下所有数据，可以分配到多个操作系统上，HDFS就是来管理存储在多台机器上文件的系统。

优点：

- 高容错性，数据保存多个副本，某个副本丢失可以自动恢复
- 适合处理大数据，能处理数据规模PB级别的数据和处理百万规模以上的文件数量
- 可构建在廉价机器上

缺点

- 不适合低延时数据访问
- 无法高效的对大量小文件进行存储
- 不支持并发写入、文件随机修改

### 架构

<img src="C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210317162117849.png" alt="image-20210317162117849" style="zoom:80%;" />

NameNode：管理者

- 管理HDFS的名称空间
- 配置副本策略
- 管理数据块（Block）映射信息
- 处理客户端读写请求

DataNode

- 存储实际的数据块
- 执行数据块的读/写操作

Client

- 文件切分，文件上传HDFS的时候，Client将文件切分成一个一个的Block，然后进行上传
- 与NameNode交互，获取文件的位置信息
- 与DataNode交互，读取或写入数据
- Client提供一些命令来管理HDFS，比如NameNode格式化
- Client可以通过一些命令来访问HDFS，比如对HDFS增删查改操作

Secondary NameNode

- 辅助NameNode，分担其工作量，如定期合并Fsimage和Edits，并推送给NameNode
- 在紧急情况下可辅助恢复NameNode

HDFS文件块大小

HDFS中的文件在物理上是分块存储，块的大小可以配置，在Hadoop2.x和3.x版本中是128M，1.X是64M。

为什么HDFS块大小默认是128M？

为什么不能远少于64MB(或128MB或256MB) （普通文件系统的数据块大小一般为4KB）减少硬盘寻道时间(disk seek time)

1.减少硬盘寻道时间

HDFS设计前提是支持大容量的流式数据操作，所以即使是一般的数据读写操作，涉及到的数据量都是比较大的。假如数据块设置过少，那需要读取的数据块就比较多，

由于数据块在硬盘上非连续存储，普通硬盘因为需要移动磁头，所以随机寻址较慢，读越多的数据块就增大了总的硬盘寻道时间。当硬盘寻道时间比io时间还要长的多时，那么硬盘寻道时间就成了系统的一个瓶颈。合适的块大小有助于减少硬盘寻道时间，提高系统吞吐量。

2.减少Namenode内存消耗

对于HDFS，他只有一个Namenode节点，他的内存相对于Datanode来说，是极其有限的。然而，namenode需要在其内存FSImage文件中中记录在Datanode中的数据块信息，假如数据块大小设置过少，而需要维护的数据块信息就会过多，那Namenode的内存可能就会伤不起了。

但是该参数也不会设置得过大。MapReduce中的map任务通常一次处理一个块中的数据，如果任务数太少（少于集群中的节点数量），作业的运行速度就会比较慢

### HDFS

#### 写流程

![image-20210317192331715](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210317192331715.png)

（1）客户端通过Distributed FileSystem 模块向 NameNode 请求上传文件，NameNode 检查目标文件是否存在，父目录是否存在

（2）NameNode 返回是否可以上传。

（3）客户端请求第一个 Block 上传到哪几个DataNode 服务器上。

（4）NameNode 返回3 个DataNode 节点，分别为dn1、dn2、dn3。

（5）客户端通过FS DataOutputStream 模块请求dn1 上传数据，dn1 收到请求会继续调用dn2，然后dn2 调用dn3，将这个通信管道建立完成。

（6）dn1、dn2、dn3 逐级应答客户端。

（7）客户端开始往dn1 上传第一个Block（先从磁盘读取数据放到一个本地内存缓存），以Packet 为单位，dn1 收到一个Packet 就会传给dn2，dn2 传给dn3；dn1 每传一个packet会放入一个应答队列等待应答。

（8）当一个Block 传输完成之后，客户端再次请求NameNode 上传第二个Block 的服务器。（重复执行3-7 步）。

副本结点选择

HDFS的存放策略是将一个副本存放在本地机架的节点上，一个副本放在同一机架的另一个节点上，最后一个副本放在不同机架的节点上。这种策略减少了机架间的数据传输，这就提高了写操作的效率

源码副本结点选择：类BlockPlacementPolicyDefault中的chooseTargetInOrder方法

#### 读流程

![image-20210317195226647](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210317195226647.png)

（1）客户端通过DistributedFileSystem 向NameNode 请求下载文件，NameNode 通过查询元数据，找到文件块所在的DataNode 地址。

（2）挑选一台DataNode（就近原则，然后随机）服务器，请求读取数据。

（3）DataNode 开始传输数据给客户端（从磁盘里面读取数据输入流，以Packet 为单位来做校验）。

（4）客户端以Packet 为单位接收，先在本地缓存，然后写入目标文件。

### NameNode和SecondaryNameNode

NN和2NN的工作机制，保证数据的可靠性，能在故障中恢复过来

![image-20210317210156227](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210317210156227.png)

fsimage文件：HDFS文件系统元数据的一个永久性检查点，其中包含HDFS文件系统的所有目录和文件inode的序列化信息

edits文件：存放HDFS文件系统的所有更新操作路径，文件系统客户端执行的所有写操作首先会被记录到edits文件中

每次NameNode启动的时候都会将Fsimage文件读入内存，加载Edits里面的更新操作，保证内存中的元数据信息是最新的、同步的，可以看成NameNode启动的时候就将Fsimage和Edits文件进行了合并

### DataNode

作用

![image-20210317210852682](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210317210852682.png)

## MapReduce

优点

- 易于编程，简单实现提供的接口，就可以完成一个分布式程序
- 良好的扩展性，可以简单增加机器扩展他的计算能力
- 高容错性，其中一台机器挂了，可以将任务转移到另一个节点上运行

缺点

- 不擅长实时计算，计算的中间结果要写磁盘
- 不擅长流式计算，mr输入数据是静态的
- 不擅长DAG（有向无环图）计算，DAG计算是指多个应用程序的输入存在依赖关系，后一个应用程序的输入是前一个的输出

框架原理

![image-20210319190717516](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210319190717516.png)

InputFormat

<img src="C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210319190955521.png" alt="image-20210319190955521" style="zoom:80%;" />

切片与MapTask并行度决定机制

FileInputFormat切片源码解析

- 遍历目录下的每一个文件（对每个文件单独切分）
- 默认情况下切分大小为BlockSize=128M
- 每次切片时，都要判断切完剩下的部分是否大于块的1.1倍（remain / blocksize <= 1.1），不大于1.1倍就划分为一块
- 只记录切片的元数据信息，比如起始位置、长度以及所在的结点列表
- 提交切片规划文件到yarn，yarn上的mr AppMaster就可以根据切片规划文件计算开启MapTask个数

CombineTextInputFormat切片机制

上述切片对每个文件进行单独切分，如果有大量小文件，会产生大量MapTask，处理效率低下，CombineTextInputFormat可以用于小文件过多的场景，可以将多个小文件从逻辑上规划到一个切片中。

MapReduce工作流程

![image-20210720203255256](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210720203255256.png)

以WordCount为例讲解执行流程

- Read阶段：将文件读入内存，默认以换行符作文分隔将每一行丢到用户编写Map函数
- Map阶段：解析每一行为key/value形式，这里就是将一行的单词提取出来，分解成<word, 1>的形式
- Collect阶段：将map生成的键值对写入到环形缓存区
- Spill（溢写）阶段：当环形缓存区之后，使用快速排序对数据进行排序，然后写到文件中
- Merge阶段：最后使用归并排序将多个文件合并成一个文件

详细流程

![image-20210319193552571](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210319193552571.png)

![image-20210319193606776](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210319193606776.png)

Shuffle机制

在Map方法之后，在Reduce方法之前的数据处理过程叫做Shuffle

![image-20210319193708928](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210319193708928.png)

Partition分区

自定义分区可以继承Partitioner类，重写getPartition()方法

- ReduceTask为1，则只会产生一个文件
- 分区数 > ReduceTask，报错
- 分区数 < ReduceTask，产生多个空文件

WritableComparable

MapTask在溢写到磁盘时会对数据进行快速排序，如果溢写了多个文件则对这些文件进行归并排序

ReduceTask在拉取多个MapTask产生的数据时会进行归并排序，都是按照数据**key**排序

自定义排序可以实现WritableComparable接口，重写里边的compareTo方法即可。

Combiner合并

- Combiner组件的父类就是Reducer
- Combiner和Reducer的区别在于运行的位置，Combiner时在每一个MapTask溢写快速排序后运行，也在归并排序多个溢写文件后运行
- Combiner能够对每一个MapTask的输出进行局部汇总，以减少网络传输量
- Combiner能够应用的前题是不能影响最终的业务逻辑，wordcount可以，但是求平均值不行

OutputFormat

<img src="C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210319195323653.png" alt="image-20210319195323653" style="zoom:80%;" />

定义输出数据的格式，默认输出格式是TextOutputFormat

ReduceTask并行度决定机制

MapTask并行度由切片个数决定，切片个数由输入文件和切片规则决定。

- ReduceTask=0，没有Reduce阶段
- ReduceTask=1，不执行分区过程
- ReduceTask数据并不是任意设置，还要考虑业务逻辑需求和集群性能，有些情况下需要计算全局汇总结果，就只能有1个ReduceTask

数据倾斜及其解决方案

Hadoop数据压缩

![image-20210319202005163](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210319202005163.png)

## Yarn

### Yarn基础架构

![image-20210320165938094](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210320165938094.png)

### Yarn工作机制

![image-20210320170005311](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210320170005311.png)

HDFS、Yarn、MapReduce三者关系

![image-20210320170431261](C:\Users\zsh\AppData\Roaming\Typora\typora-user-images\image-20210320170431261.png)

Yarn作业提交过程

Yarn调度器和调度算法

- FIFO调度器
- 容量调度器（Capacity Scheduler）
- 公平调度器（Fair Schedule）

FIFO调度器

单队列，根据提交作业的先后顺序，先来先服务

容量调度器

公平调度器

## 组件间的通信机制-RPC

