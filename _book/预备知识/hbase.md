# hbase学习笔记

### 第一部分.HBase架构、读写原理及刷写合并切分机制
#### 一、HBase的结构

##### （一）逻辑结构

![1573003062207](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1573003062207.png)

HBase以列族为纵向分隔单位进行存储，每个列族为一个store。

一个表可以包含若干region。

Regin以rowkey的大小范围进行分割，其中每个region都包含所有列族的信息。

Rowkey有序，以字典序进行排列，非空，不重复，通过Rowkey获取特定数据。

##### （二）物理存储结构

![1573003085462](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1573003085462.png)

数据模型：

1）Name Space（在0.96版本以后出现）

命名空间，类似于关系型数据库的 DatabBase 概念，每个命名空间下有多个表。 HBase有两个自带的命名空间，分别是 hbase 和 default， hbase 中存放的是 HBase 内置的表，如meta表，default 表是用户默认使用的命名空间。

2） Region

类似于关系型数据库的表概念。不同的是， HBase 定义表时只需要声明列族即可，不需要声明具体的列。这意味着，往 HBase 写入数据时，字段可以动态、按需指定。因此，和关系型数据库相比， HBase 能够轻松应对字段变更的场景。

3） Row

HBase 表中的每行数据都由一个 RowKey 和多个 Column（列）组成，数据是按照 RowKey的字典顺序存储的，并且查询数据时只能根据 RowKey 进行检索，所以 RowKey 的设计十分重要。

4） Column

HBase 中的每个列都由 Column Family(列族)和 Column Qualifier（列限定符） 进行限定，例如 info： name， info： age。建表时，只需指明列族，而列限定符无需预先定义。

5） Time Stamp

用于标识数据的不同版本（version）， 每条数据写入时， 如果不指定时间戳，系统会自动为其加上该字段，其值为写入 HBase 的时间。单元格内不同版本的值按时间倒序排列，最新的数据排在最前面。

6） Cell

由RowKey、列族、列限定符、timestamp唯一定位 ，Cell中存放一个值（Value）和一个版本号。

 

 

 

 

**HBase表特点：**

* 数据规模大，单表可容纳数十亿行，上百万列。

* 无模式，不像关系型数据库有严格的Scheme，每行可以有任意多的列，列可以动态增加，不同行可以有不同的列，列的类型没有限制。

* 稀疏，值为空的列不占存储空间，表可以非常稀疏，但实际存储时，能进行压缩。

* 面向列族，面向列族的存储和权限控制，支持列族独立查询。

* 数据多版本，利用时间戳来标识版本

* 数据无类型，所有数据以字节数据形式存储



#### 二、HBase的架构

   ![1573003294840](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1573003294840.png)

##### 架构角色：

**1**）**Region Server**

Region Server 为 Region 的管理者，一个RegionServer里有多个Region，一个RegionServer管理着多个Region，在HBase运行期间，可以动态添加、删除HRegionServer。

处理Client端的读写请求（根据从HMaster返回的元数据找到对应的Region来读写数据）。

RegionServer维护一个HLog。 

管理Region的Split分裂、StoreFile的Compaction合并。

**2**）**Master**

负责管理HBase元数据，即表的结构、表存储的Region等元信息。

负责表的创建，删除和修改（因为这些操作会导致HBase元数据的变动）。

负责为HRegionServer分配Region，分配好后也会将元数据写入相应位置，同时负责监控每个 RegionServer的状态，负载均衡和故障转移。

如果对可用性要求较高，它需要做HA高可用（通过Zookeeper）。但是HMaster不会去处理Client端的数据读写请求，因为这样会加大其负载压力，具体的读写请求它会交给HRegionServer来做。

**3**）**Zookeeper**

HBase 通过 Zookeeper 来做 Master 的高可用、 RegionServer 的监控、元数据的入口以及集群配置的维护等工作。

**4**）**HDFS**

HDFS 为 HBase 提供最终的底层数据存储服务，同时为 HBase 提供高可用的支持。

##### RegionServer内部结构：

**1**）**HRegion** 

​        一个HRegion里可能有1个或多个Store。 

HRegion是分布式存储和负载的最小单元。 

一张表通常被保存在多个HRegionServer的多个Region中。 

因为HBase用于存储海量数据，故一张表中数据量非常之大，单机一般存不下这么大的数据，故HBase 会将一张表按照RowKey范围将大表划分为多个Region，每个Region保存表的一段连续数据。初始只有1个 Region，当一个Region增大到某个阈值后，便分割为两个。

**2**）**Store**

Store是存储落盘的最小单元，由内存中的MemStore和磁盘中的若干StoreFile组成。

一个Store里有1个或多个StoreFile和一个memStore。

每个Store存储一个列族。

**3**）**MemStore**

写缓存，由于 HFile 中的数据要求是有序的， 所以数据是先存储在 MemStore 中，排好序后，等到达刷写时机才会刷写到 HFile，每次刷写都会形成一个新的 HFile。

**4**）**StoreFile**

保存实际数据的物理文件， StoreFile 以 HFile 的形式存储在 HDFS 上。

**5**）**WAL（HLog）**

由于数据要经 MemStore 排序后才能刷写到 HFile， 但把数据保存在内存中会有很高的概率导致数据丢失，为了解决这个问题，数据会先写在一个叫做 Write-Ahead logfile 的文件中，然后再写入 MemStore 中。所以在系统出现故障的时候，数据可以通过这个日志文件重建。





#### 三、HBase读写数据流程

##### 1.写过程

   ![1573003431862](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1573003431862.png)

1） Client 先访问 zookeeper，获取 hbase:meta 表位于哪个 Region Server中，并将该位置信息写入Client Cache。

（详细定位region过程参考文件“hbase~查找region问题”）

（注：为了加快数据访问速度，将元数据、Region位置等信息缓存在Client Cache中。）

2）访问对应的 Region Server，获取hbase:meta 表，根据写请求的

namespace:table/rowkey，查询出目标数据位于哪个 Region Server 中的哪个 Region 中。并将该 table 的 region 信息缓存在Client cache，方便下次访问。

3）与目标 Region Server 进行通讯；

4） HRegionServer先将操作和数据写入HLog（预写日志，Write Ahead Log，WAL），再将数据写入MemStore，并保持有序。

5）向客户端发送 ack；

6）当MemStore的数据量超过阈值时，将数据刷写到磁盘，生成一个StoreFile文件。

当Store中StoreFile的数量超过阈值时，将若干小StoreFile合并（Compact）为一个大StoreFile。 

​        当Region中最大Store的大小超过阈值时，Region分裂（Split），等分成两个子Region。



##### 2.读过程

   ![1573003464698](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1573003464698.png)

1） Client 先访问 zookeeper，获取 hbase:meta 表位于哪个Region Server中，并将该位置信息写入Client Cache。

（具体定位region过程参考文件“hbase~查找region问题”）

（注：为了加快数据访问速度，将元数据、Region位置等信息缓存在Client Cache中。）

2）访问对应的 Region Server，获取hbase:meta 表，根据读请求的

namespace:table/rowkey，查询出目标数据位于哪个Regi on Server中的哪个Region中。并将该 table的region信息缓存在Client cache，方便下次访问。

3）与目标 Region Server 进行通讯；

4）HRegionServer先从MemStore读取数据，如未找到，在检查blockcache看是否包含该行的block是否最近被访问过，最后再从StoreFile中读取。此处所有数据是指同一条数据的不同版本（time stamp）或者不同的类型（Put/Delete）。

5）将从磁盘文件中查询到的数据块（Block， HFile 数据存储单元，默认大小为 64KB）缓存到Block Cache。

6）将查询到的结果合并后返回到客户端。





#### 四、HBase对Region的拆分以及对StoreFile的合并

##### 1.对region的拆分

默认情况下，每个 Table 起初只有一个 Region，随着数据的不断写入， Region 会自动进行拆分。刚拆分时，两个子 Region 都位于当前的Region Server，但处于负载均衡的考虑，HMaster 有可能会将某个 Region 转移给其他的 Region Server。

Region Split 时机：

* 当1个 region中的某个 Store下所有 StoreFile的总大小超过hbase.hregion.max.filesize，该 Region 就会进行拆分（0.94 版本之前）。

* 当1个region中的某个Store下所有StoreFile的总大小超过Min(R^2 * "hbase.hregion.memstore.flush.size",hbase.hregion.max.filesize)，该 Region 就会进行拆分，其中R为当前RegionServer中属于该Table的个数（0.94 版本之后）。

##### 2.对StoreFile的合并

由于 memstore每次刷写都会生成一个新的 HFile，且同一个字段的不同版本（timestamp）和不同类型（Put/Delete）有可能会分布在不同的 HFile 中，因此查询时需要遍历所有的 HFile。为了减少 HFile 的个数，以及清理掉过期和删除的数据，会进行 StoreFile Compaction，以便提高数据读取效率。

   ![1573003555652](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1573003555652.png)

Compaction分为两种：

* major compaction 

​        将Store下面**所有**StoreFile合并为一个StoreFile，此操作会删除过期和删除的数据

* minor compaction 

​        选取Store下的**部分**StoreFile，将它们合并为一个StoreFile，此操作不会清理过期和删除的数据。

合并的触发时机：

**参考url：https://www.jianshu.com/p/b2d8a8c157d8 **

​	HBase中可以触发compaction的因素有很多，最常见的因素有这么三种：Memstore Flush、后台线程周期性检查、手动触发。

* **Memstore Flush: **

应该说compaction操作的源头就来自flush操作，memstore flush会产生HFile文件，文件越来越多就需要compact。因此在每次执行完Flush操作之后，都会对当前Store中的文件数进行判断，一旦文件数 > 预值 ，就会触发compaction。需要说明的是，compaction都是以Store为单位进行的，而在Flush触发条件下，整个Region的所有Store都会执行compact，所以会在短时间内执行多次compaction。

* **后台线程周期性检查：**

后台线程CompactionChecker定期触发检查是否需要执行compaction，检查周期为：hbase.server.thread.wakefrequencyhbase.server.compactchecker.interval.multiplier。和Memstore flush不同的是，该线程优先检查文件数＃是否大于，一旦大于就会触发compaction。如果不满足，它会接着检查是否满足major compaction条件，简单来说，如果当前store中hfile的最早更新时间早于某个值mcTime，就会触发major compaction，HBase预想通过这种机制定期删除过期数据。上文mcTime是一个浮动值，浮动区间默认为［7-70.2，7+7*0.2］，其中7为hbase.hregion.majorcompaction，0.2为hbase.hregion.majorcompaction.jitter，可见默认在7天左右就会执行一次major compaction。用户如果想禁用major compaction，只需要将参数hbase.hregion.majorcompaction设为0。

* **手动触发：**

一般来讲，手动触发compaction通常是为了执行major compaction，原因有三，其一是因为很多业务担心自动major compaction影响读写性能，因此会选择低峰期手动触发；其二也有可能是用户在执行完alter操作之后希望立刻生效，执行手动触发major compaction；其三是HBase管理员发现硬盘容量不够的情况下手动触发major compaction删除大量过期数据；无论哪种触发动机，一旦手动触发，HBase会不做很多自动化检查，直接执行合并。



#### 五、Memstore刷写时机

***参考博客url：https://blog.csdn.net/shenshouniu/article/details/84592496***

**1. Memstore的作用**

一个Store中总会有一个Memstore和多个HFile，每一次刷写就会生成一个HFile。

MemStore的实现目的不是加速数据的写入，主要是维持HBase中的数据按照rowkey顺序来存储，所以先使用MemStore先对数据进行整理排序后再持久化到HDFS。

**2. Memstore刷写时机**

（1）数据刷写时机一（`hbase.hregion.memstore.flush.size`）

当Memstore占用的内存大小达到`hbase.hregion.memstore.flush.size`的配置值的时候就会触发一次刷写，生产一个HFile。

**因为MemStore的刷写存在一个定期检查时间，有时候可能数据增长速度太快，在还未达到检查时间之前，数据就达到了hbase.hregion.memstore.flush.size的好几倍，从而被阻塞住了。这种情况在下面讨论。**

（2）数据刷写时机二 （globalMemStoreSize）

`globalMemStoreLimitLowMarkPercent`:全局的memstore刷写下限，过去通过配置`hbase.regionserver.global.memstore.lowerLimit`来定义，现在统一改成：`hbase.regionserver.global.memstore.size.lower.limit`。该配置项是一个百分比，所以取值在0.0-1.0，默认是0.95。

globalMemStoreSize表示全局memstore容量，这个值计算方法如下：hbase_heapsize(Regionserver 占用堆内存大小)*`hbase.regionserver.global.memstore.size`

`hbase.regionserver.global.memstore.size`的默认值是0.4。

一旦达到这个阈值：

`regionserverHeapSize*hbase.regionserver.global.memstore.size*hbase.regionserver.global.memstore.size.lower.limit`

就会触发一次强制的刷写。

（3）数据刷写时机三 （WAL的数量大于maxLogs）

WAL的数量大于maxLogs时，也会触发一次刷写，不过不会发生阻塞事件,倒是会警告一下。该参数其实新版本已经不需要进行设置了，最大就是32，也可以更小，根据`hbase.regionserver.global.memstore.size`来决定：

`Math.max(32，(regionserverHeapSize*memstoreSizeRatio*2/logRollSize))`

（4）数据刷写时机四(自动刷写间隔)

`hbase.regionserver.optionalcacheflushinterval`：表示memstore的刷写间隔，默认值是3600000，即1个小时。如果设定为0，则意味着关闭自动刷写。

（5）数据刷写时机五（手动刷写）

* API:

* * flush(TableName tableName):对单表进行刷写。

* * flushRegion(byte[] regionName):对单个Region进行刷写。

* HBase Shell:

* * `flush ‘tablename’`

* * `flush ‘regionname’`

**3.Memstore阻塞的情况**

（1）memstore阻塞情况一（数据增长速度太快）

`hbase.hregion.memstore.flush.size` 默认阈值是128MB

`hbase.hregion.memstore.block.multiplier`:是一个倍数，默认是4。

上面两个数的乘积默认为512M，因为MemStore的刷写存在一个定期检查时间，在下一次刷写检查到来之前若达到了这个阈值，就会立即触发刷写，同时阻塞住所有的写入该Store的写请求。

（2）memstore阻塞情况二（global.memstore.size）

当`hbase_heapsize`(Regionserver占用堆内存大小)*`hbase.regionserver.global.memstore.size`大小达到阈值时，就会阻塞整个HBase集群的写入。

 

------------------------------------





### 第二部分.HBase的性能优化

##### HBase性能优化

##### 1.设置HBase高可用

在 HBase 中 HMaster 负责监控 HRegionServer 的生命周期，均衡 RegionServer 的负载，如果 HMaster 挂掉了，那么整个 HBase 集群将陷入不健康的状态，并且此时的工作状态并不会维持太久。所以 HBase 支持对 HMaster 的高可用配置。

Hbase本身就默认支持对HMater的高可用，只不过需要手动启用备用master。

<1>在需要启动master的节点上使用如下命令：

​     [root@node02 hbase]# bin/hbase-daemon.sh start master

​     访问网址：node01:16010      ;   node02:16010

​     即可查看到node02为备用master

<2>上一种方法每次都需要手动启用，可以采用如下配置，即可在启动HBase时，自动启动备份Master：

（1）关闭 HBase 集群（如果没有开启则跳过此步）

​     \# bin/stop-hbase.sh

（2）在 conf 目录下创建 backup-masters 文件

​     \#touch conf/backup-masters

（3）在 backup-masters 文件中配置高可用 HMaster 节点

​     \#echo node02 > conf/backup-masters

（4）将整个 conf 目录复制到其他节点

（5）测试

##### 2.预分区

**参考博客地址：https://blog.51cto.com/12445535/2352410**

​	默认情况下，在创建HBase表的时候会自动创建一个region分区，当导入数据的时候，所有的HBase客户端都向这一个region写数据，直到这个region足够大了才进行切分。一种可以加快批量写入速度的方法是通过预先创建一些空的regions，这样当数据写入HBase时，会按照region分区情况，在集群内做数据的负载均衡。

​	每一个 region 维护着 StartRow 与 EndRow，如果加入的数据符合某个 Region 维护的RowKey 范围，则该数据交给这个 Region 维护。那么依照这个原则，我们可以将数据所要投放的分区提前大致的规划好，以提高 HBase 性能。

预分区的设计与实际集群大小以及实际数据量的大小相关。同时预分区的分区键的选择也与rowkey的定义方法相关。

​      ***常用的预分区的两种方式：***

（1）Shell命令行执行预分区

​      在命令行中进行预分区的命令：

​	a)  直接在命令行中指定分区键

`hbase(main):001:0> create 'testFenqu','info',SPLITS=>['1000','2000','3000']`

​      在页面中查看，可以看到分成了四个区，第一个区的rowkey范围是[空，1000]，第二个是[1000,2000]，第三个是[3000,4000]，第四个是[4000，空]

![img](file:///C:/Users/lv/AppData/Local/Temp/msohtmlclip1/01/clip_image002.jpg)

​			b)  将分区键写入文件，通过读文件进行分区

​     		文件内容：

![img](file:///C:/Users/lv/AppData/Local/Temp/msohtmlclip1/01/clip_image004.jpg)

​            SHELL命令：

​			`hbase(main):001:0> create 'testFenqu2','info',SPLITS_FILE=>'splits.txt'`

​     在页面中查看，可以看到分成了四个区，第一个区的rowkey范围是 [空，aa]，第二个是[aa，bb]，第三个是[bb，cc]，第四个是[cc，dd]，第五个是 [dd，空]。

![img](file:///C:/Users/lv/AppData/Local/Temp/msohtmlclip1/01/clip_image006.jpg)

（2）调用API在程序创建表的时候进行预分区

参考博客：https://blog.csdn.net/qq_36864672/article/details/81019748



#####  3.合理设计RowKey

HBase中RowKey用来检索表中的记录，支持以下三种方式：

* 通过单个row key访问：即按照某个row key键值进行get操作；

* 通过RowKey的range进行scan：即通过设置startRowKey和endRowKey，在这个范围内进行扫描；

* 全表扫描：即直接扫描整张表中所有行记录。

在HBase中，RowKey可以是任意字符串，最大长度64KB，实际应用中一般为10~100bytes，存为byte[]字节数组，一般设计成定长的。

row key是按照字典序存储，因此，设计row key时，要充分利用这个排序特点，将经常一起读取的数据存储到一块，将最近可能会被访问的数据放在一块。

一条数据的唯一标识就是 RowKey，那么这条数据存储于哪个分区，取决于 RowKey 处于哪个一个预分区的区间内，设计 RowKey 的主要目的 ，就是让数据均匀的分布于所有的region 中，在一定程度上防止数据倾斜。

以上两种目的相互矛盾，因此根据实际的应用需求进行合理的rowkey设计，既要考虑到散列性，又要考虑到集中性。

常用的设置RowKey的方法：

（1）生成随机数、hash、散列值

（2）字符串反转

（3）字符串拼接----将时间戳作为RowKey的一部分

一个设计rowkey的例子：

​	假设用hbase存储移动公司存储用户的通话记录，分区数300个。分区键为000 001 002 … 299。 

​	用户rowkey可以设计为：

​		分区键拼接上手机号+年份+月份+其他信息

​		如：xxx_13566668888_2019-07

​		前边的xxx分区键可以计算得到，计算公式可以为：

​		[（13566668888+201907）.hash]%300

​		得出的rowKey可能为：

​		000_13566668888_2019-07-16 12：12：12

通过上述的这种设计，可以实现一个手机号的同月份的通话记录存储在同一个分区内，不同手机号的通话记录通过取模运算存储在不同的分区，既考虑了散列性，又考虑了集中性。

##### 4.内存优化

HBase 操作过程中需要大量的内存开销，毕竟 Table 是可以缓存在内存中的，一般会分配整个可用内存的 70%给 HBase 的 Java 堆。但是不建议分配非常大的堆内存，因为 GC 过程持续太久会导致 RegionServer 处于长期不可用状态，一般 16~48G 内存就可以了，如果因为框架占用内存过高导致系统内存不足，框架一样会被系统服务拖死。

##### 5.基础优化

（1）允许在 HDFS 的文件中追加内容

hdfs-site.xml、 hbase-site.xml

属性： dfs.support.append

解释：开启 HDFS 追加同步，可以优秀的配合 HBase 的数据同步和持久化。默认值为 true。

（2）优化 DataNode 允许的最大文件打开数

​        hdfs-site.xml

​        属性： dfs.datanode.max.transfer.threads

​        解释： HBase 一般都会同一时间操作大量的文件，根据集群的数量和规模以及数据动作，设置为 4096 或者更高。默认值： 4096

（3）优化延迟高的数据操作的等待时间

hdfs-site.xml

属性： dfs.image.transfer.timeout

解释：如果对于某一次数据操作来讲，延迟非常高， socket 需要等待更长的时间，建议把该值设置为更大的值（默认 60000 毫秒），以确保 socket 不会被 timeout 掉。

（4）优化数据的写入效率

​        mapred-site.xml

​        属性：

​        mapreduce.map.output.compress

​        mapreduce.map.output.compress.codec

​        解释：开启这两个数据可以大大提高文件的写入效率，减少写入时间。第一个属性值修改为true，第二个属性值修改为：org.apache.hadoop.io.compress.GzipCodec 或者其他压缩方式。

（5）设置 RPC 监听数量

​        hbase-site.xml

属性： Hbase.regionserver.handler.count

解释：默认值为 30，用于指定 RPC 监听的数量，可以根据客户端的请求数进行调整，读写请求较多时，增加此值

（6）优化 HStore 文件大小

​        hbase-site.xml

​        属性： hbase.hregion.max.filesize

​        解释：默认值 10737418240（10GB），如果需要运行 HBase 的 MR 任务，可以减小此值，因为一个 region 对应一个 map 任务，如果单个 region 过大，会导致 map 任务执行时间过长。该值的意思就是，如果 HFile 的大小达到这个数值，则这个 region 会被切分为两个 Hfile。

（7）优化 HBase 客户端缓存

​      hbase-site.xml

​        属性： hbase.client.write.buffer

​        解释：用于指定 Hbase 客户端缓存，增大该值可以减少 RPC 调用次数，但是会消耗更多内存，反之则反之。一般我们需要设定一定的缓存大小，以达到减少 RPC 次数的目的。

（8）指定 scan.next 扫描 HBase 所获取的行数

​      hbase-site.xml

​        属性： hbase.client.scanner.caching

​        解释：用于指定 scan.next 方法获取的默认行数，值越大，消耗内存越大。

（9）flush、 compact、 split 机制

​      当 MemStore 达到阈值，将 Memstore 中的数据 Flush 进 Storefile； compact 机制则是把 flush出来的小文件合并成大的 Storefile 文件。 split 则是当 Region 达到阈值，会把过大的 Region一分为二。

​        建议：关闭线上自动split操作。

这一方面可以避免’拆分合并风暴’，当用户的 region 大小以恒定的速度保持增长时，或者在某些巧合的场景下，大量 region 可能会在同一时间发生split，因为这个过程会重写拆分之后的 region ，这将引起磁盘IO上升。手动拆分可以控制执行时间，在不同的region上交互执行，这样可以尽可能分散IO压力，避免风暴。

另一方面，手动拆分可以控制哪些 region 可用。试想，某天运维人员碰到一个问题想对问题现场进行跟踪，转眼没看，发现 region 已经被拆分了，现场被毁了，那种感觉岂不是很不好。手动拆分就可以避免这种奇葩场景。

实践方法：可以将配置文件中‘hbase.hregion.max.filesize’设置为一个较大的值（比如200G）, 这样系统就不会触发自动split操作。

---------------------

### 第三部分.HBase的Shell操作


hbase | shell命令 |  描述  
 :-: | :-: | :-: 
create	|创建表|< create ‘表名’, ‘列族名’, ‘列族名2’,‘列族名N’ >
list	|查看所有表	|< list all >
describe	|显示表详细信息	|< describe ‘表名’ >
exists	|判断表是否存在	|< exists ‘表名’ >
enable	|使表有效	|< enable ‘表名’ >
disable	|使表无效	|< disable ‘表名’ >
is_enabled	|判断是否启动表	|< is_enabled ‘表名’ >
is_disabled	|判断是否禁用表	|< is_disabled ‘表名’ >
count	|统计表中行的数量	|< count ‘表名’ >
put	|添加记录	|< put ‘表名’, ‘row key’, ‘列族1 : 列’, ‘值’ >
get	|获取记录(row key下所有)	|< get ‘表名’, ‘row key’>
get	|获取记录(某个列族)	|< get ‘表名’, ‘row key’, ‘列族’>
get	|获取记录(某个列)	|< get ‘表名’,‘row key’,‘列族:列’ >
delete	|删除记录	|< delete ‘表名’, ‘row key’, ‘列族:列’ >
deleteall	|删除一行	|< deleteall ‘表名’,‘row key’>
drop	|删除表	|<disable ‘表名’> < drop ‘表名’>
alter	|修改列族（column family）	
incr	|增加指定表，行或列的值	
truncate	|清空表	|逻辑为先删除后创建 <truncate ‘表明’>
scan	|通过对表的扫描来获取对用的值	|<scan ‘表名’>
tools	|列出hbase所支持的工具	
status	|返回hbase集群的状态信息	
version	|返回hbase版本信息	
exit	|退出hbase shell	
shutdown	|关闭hbase集群(与exit不同)