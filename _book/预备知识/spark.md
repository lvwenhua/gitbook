# spark学习笔记

## Spark架构、原理、运行模式

## 一、Spark的组成

![1571625935142](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571625935142.png)

它的主要组件有：

* **SparkCore**：实现了 Spark 的基本功能，包含任务调度、内存管理、错误恢复、与存储系统交互等模块。 Spark Core 中还包含了对弹性分布式数据集(Resilient Distributed DataSet，简称 RDD)的 API 定义。 并为运行在其上的上层组件提供API。
* **SparkSQL**：是 Spark用来操作结构化数据的程序包。通过Spark SQL，我们可以使用 SQL或者 Apache Hive 版本的 SQL 方言(HQL)来查询数据。Spark SQL 支持多种数据源，比如 Hive表、 Parquet 以及 JSON 等。
* **SparkStreaming**：是 Spark 提供的对实时数据进行流式计算的组件。提供了用来操作数据流的 API，并且与 Spark Core 中的 RDD API 高度对应。 
* **MLlib**：提供常见的机器学习(ML)功能的程序库。包括分类、回归、聚类、协同过滤等，还提供了模型评估、数据 导入等额外的支持功能 。
* **GraphX**：提供一个分布式图计算框架，能高效进行图计算。
* **集群管理器**： Spark 设计为可以高效地在一个计算节点到数千个计算节点之间伸缩计算。为了实现这样的要求，同时获得最大灵活性， Spark 支持在各种集群管理器(ClusterManager)上运行，包括 Hadoop YARN、 Apache Mesos，以及 Spark 自带的一个简易调度器，叫作独立调度器。 

## 二、Spark基本运行流程

### 2.1 Spark中的基本概念

（1）Application：表示用户编写的Spark应用程序

（2）Driver：表示main()函数，创建SparkContext。由SparkContext负责与ClusterManager通信，进行资源的申请，任务的分配和监控等。程序执行完毕后关闭SparkContext

（3）Executor：某个Application运行在Worker节点上的一个进程，该进程负责运行某些task，并且负责将数据存在内存或者磁盘上。在Spark on Yarn模式下，其进程名称为 CoarseGrainedExecutor Backend，一个CoarseGrainedExecutor Backend进程有且仅有一个executor对象，它负责将Task包装成taskRunner，并从线程池中抽取出一个空闲线程运行Task，这样，每个CoarseGrainedExecutorBackend能并行运行Task的数据就取决于分配给它的CPU的个数。

（4）Worker：集群中可以运行Application代码的节点。在Standalone模式中指的是通过slave文件配置的worker节点，在Spark on Yarn模式中指的就是NodeManager节点。

（5）Task：在Executor进程中执行任务的工作单元，多个Task组成一个Stage

（6）Job：包含多个Task组成的并行计算，是由Action行为触发的

（7）Stage：每个Job会被拆分很多组Task，作为一个TaskSet，其名称为Stage

（8）DAGScheduler：根据Job构建基于Stage的DAG，并提交Stage给TaskScheduler，其划分Stage的依据是RDD之间的依赖关系

（9）TaskScheduler：将TaskSet提交给Worker（集群）运行，每个Executor运行什么Task就是在此处分配的。




### 2.2 运行架构

Spark运行架构包括集群资源管理器（ClusterManager）、运行作业任务的工作节点（Worker Node）、每个应用的任务控制节点（Driver）和每个工作节点上负责具体任务的执行进程（Executor）。

![1571628242857](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571628242857.png)

- 一个应用由一个Driver和若干个作业构成，一个作业由多个阶段构成，一个阶段由多个没有Shuffle关系的任务组成
- 当执行一个应用时，Driver会向集群管理器申请资源，启动Executor，并向Executor发送应用程序代码和文件，然后在Executor上执行任务，运行结束后，执行结果会返回给Driver，或者写到HDFS或者其他数据库中。

![1571628196854](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571628196854.png)

### 2.3 运行流程图



![1571627533454](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571627533454.png)

### 2.4 流程简要说明

(1) 构建Spark Application的运行环境（启动SparkContext），SparkContext向资源管理器（可以是Standalone、Mesos或YARN）注册并申请运行Executor资源；

(2) 资源管理器分配Executor资源并启动StandaloneExecutorBackend，Executor运行情况将随着心跳发送到资源管理器上；

(3) SparkContext构建成DAG图，将DAG图分解成Stage，并把Taskset发送给Task Scheduler。Executor向SparkContext申请Task

(4) Task Scheduler将Task发放给Executor运行同时SparkContext将应用程序代码发放给Executor。

(5) Task在Executor上运行，运行完毕释放所有资源。

### 2.5 Spark运行架构特点

（1）每个Application获取专属的executor进程，该进程在Application期间一直驻留，并以多线程方式运行tasks。这种Application隔离机制有其优势的，无论是从调度角度看（每个Driver调度它自己的任务），还是从运行角度看（来自不同Application的Task运行在不同的JVM中）。当然，这也意味着Spark Application不能跨应用程序共享数据，除非将数据写入到外部存储系统。

（2）Spark与资源管理器无关，只要能够获取executor进程，并能保持相互通信就可以了。

（3）提交SparkContext的Client应该靠近Worker节点（运行Executor的节点)，最好是在同一个Rack里，因为Spark Application运行过程中SparkContext和Executor之间有大量的信息交换；如果想在远程集群中运行，最好使用RPC将SparkContext提交给集群，不要远离Worker运行SparkContext。

（4）Task采用了数据本地性和推测执行的优化机制。

### 2.6 重要角色

#### 2.6.1 Driver（驱动器） 

Spark的驱动器是执行开发程序中的 main 方法的进程。它负责开发人员编写的用来创建SparkContext、创建 RDD，以及进行 RDD 的转化操作和行动操作代码的执行。如果用 spark shell，那么当启动Spark shell的时候，系统后台自启了一个Spark驱动器程序，就是在 Spark shell 中预加载的一个叫作sc的SparkContext对象。如果驱动器程序终止，那么 Spark 应用也就结束了。 主要负责： 

1. 把用户程序转为作业(job)
2. 跟踪Executor的运行状况
3. 为执行器节点调度任务
4.  UI 展示应用运行状况 

#### 2.6.2 Executor（执行器） 

Executor是spark里的进程模型，可以套用到不同的资源管理系统上，与`SchedulerBackend`配合使用。负责在 Spark 作业中运行Task，任务间相互独立。Spark应用启动时， Executor 节点被同时启动，并且始终伴随着整个 Spark 应用的生命周期而存在。如果有 Executor 节点发生了故障或崩溃， Spark 应用也可以继续执行，会将出错节点上的任务调度到其他 Executor 节点上继续运行。

内部有个线程池，有个running tasks map，有个actor，接收由`SchedulerBackend`发来的事件。

**主要负责：**

1. 负责运行组成 Spark 应用的任务，并将结果返回给驱动器进程；
2. 通过自身的块管理器（Block Manager）为用户程序中要求缓存的 RDD 提供内存式存储。 RDD 是直接缓存在 Executor 进程内的，因此任务可以在运行时充分利用缓存数据加速运算。  

**事件处理**

1. `launchTask`。根据task描述，生成一个`TaskRunner`线程，丢尽running tasks map里，用线程池执行这个`TaskRunner`
2. `killTask`。从running tasks map里拿出线程对象，调它的kill方法。

#### 2.6.3 DAGScheduler

Job=多个stage，Stage=多个同种task, Task分为ShuffleMapTask和ResultTask，Dependency分为ShuffleDependency和NarrowDependency

面向stage的切分，切分依据为宽依赖

维护waiting jobs和active jobs，维护waiting stages、active stages和failed stages，以及与jobs的映射关系

**主要职能：**

> 1、接收提交Job的主入口，`submitJob(rdd, ...)`或`runJob(rdd, ...)`。在`SparkContext`里会调用这两个方法。 
>
>   -  生成一个Stage并提交，接着判断Stage是否有父Stage未完成，若有，提交并等待父Stage，以此类推。结果是：DAGScheduler里增加了一些waiting stage和一个running stage。
>   -  running stage提交后，分析stage里Task的类型，生成一个Task描述，即TaskSet。
>   -  调用`TaskScheduler.submitTask(taskSet, ...)`方法，把Task描述提交给TaskScheduler。TaskScheduler依据资源量和触发分配条件，会为这个TaskSet分配资源并触发执行。
>   - `DAGScheduler`提交job后，异步返回`JobWaiter`对象，能够返回job运行状态，能够cancel job，执行成功后会处理并返回结果
>
> 2、处理`TaskCompletionEvent` 
>
> - - 如果task执行成功，对应的stage里减去这个task，做一些计数工作： 
>     - 如果task是ResultTask，计数器`Accumulator`加一，在job里为该task置true，job finish总数加一。加完后如果finish数目与partition数目相等，说明这个stage完成了，标记stage完成，从running stages里减去这个stage，做一些stage移除的清理工作
>     - 如果task是ShuffleMapTask，计数器`Accumulator`加一，在stage里加上一个output location，里面是一个`MapStatus`类。`MapStatus`是`ShuffleMapTask`执行完成的返回，包含location信息和block size(可以选择压缩或未压缩)。同时检查该stage完成，向`MapOutputTracker`注册本stage里的shuffleId和location信息。然后检查stage的output location里是否存在空，若存在空，说明一些task失败了，整个stage重新提交；否则，继续从waiting stages里提交下一个需要做的stage
>   - 如果task是重提交，对应的stage里增加这个task
>   - 如果task是fetch失败，马上标记对应的stage完成，从running stages里减去。如果不允许retry，abort整个stage；否则，重新提交整个stage。另外，把这个fetch相关的location和map任务信息，从stage里剔除，从`MapOutputTracker`注销掉。最后，如果这次fetch的blockManagerId对象不为空，做一次`ExecutorLost`处理，下次shuffle会换在另一个executor上去执行。
>   - 其他task状态会由`TaskScheduler`处理，如Exception, TaskResultLost, commitDenied等。
>
> 3、其他与job相关的操作还包括：cancel job， cancel stage, resubmit failed stage等
>
>  
>

#### 2.6.4 TaskScheduler

维护task和executor对应关系，executor和物理资源对应关系，在排队的task和正在跑的task。

内部维护一个任务队列，根据FIFO或Fair策略，调度任务。

`TaskScheduler`本身是个接口，spark里只实现了一个`TaskSchedulerImpl`，理论上任务调度可以定制。

主要功能：

> `1、submitTasks(taskSet)`，接收`DAGScheduler`提交来的tasks 
>
> - - 为tasks创建一个`TaskSetManager`，添加到任务队列里。`TaskSetManager`跟踪每个task的执行状况，维护了task的许多具体信息。
>   - 触发一次资源的索要。 
>     - 首先，`TaskScheduler`对照手头的可用资源和Task队列，进行executor分配(考虑优先级、本地化等策略)，符合条件的executor会被分配给`TaskSetManager`。
>     - 然后，得到的Task描述交给`SchedulerBackend`，调用`launchTask(tasks)`，触发executor上task的执行。task描述被序列化后发给executor，executor提取task信息，调用task的`run()`方法执行计算。
>
> `2、cancelTasks(stageId)`，取消一个stage的tasks 
>
> - - 调用`SchedulerBackend`的`killTask(taskId, executorId, ...)`方法。taskId和executorId在`TaskScheduler`里一直维护着。
>
> ```
> 3、resourceOffer(offers: Seq[Workers])`，这是非常重要的一个方法，调用者是`SchedulerBacnend`，用途是底层资源`SchedulerBackend`把空余的workers资源交给`TaskScheduler`，让其根据调度策略为排队的任务分配合理的cpu和内存资源，然后把任务描述列表传回给`SchedulerBackend
> ```
>
> - - 从worker offers里，搜集executor和host的对应关系、active executors、机架信息等等
>   - worker offers资源列表进行随机洗牌，任务队列里的任务列表依据调度策略进行一次排序
>   - 遍历每个taskSet，按照进程本地化、worker本地化、机器本地化、机架本地化的优先级顺序，为每个taskSet提供可用的cpu核数，看是否满足 
>     - 默认一个task需要一个cpu，设置参数为`"spark.task.cpus=1"`
>     - 为taskSet分配资源，校验是否满足的逻辑，最终在`TaskSetManager`的`resourceOffer(execId, host, maxLocality)`方法里
>     - 满足的话，会生成最终的任务描述，并且调用`DAGScheduler`的`taskStarted(task, info)`方法，通知`DAGScheduler`，这时候每次会触发`DAGScheduler`做一次`submitMissingStage`的尝试，即stage的tasks都分配到了资源的话，马上会被提交执行
>
> `4、statusUpdate(taskId, taskState, data)`,另一个非常重要的方法，调用者是`SchedulerBacnend`，用途是`SchedulerBacnend`会将task执行的状态汇报给`TaskScheduler`做一些决定 
>
> - - 若`TaskLost`，找到该task对应的executor，从active executor里移除，避免这个executor被分配到其他task继续失败下去。
>- task finish包括四种状态：finished, killed, failed, lost。只有finished是成功执行完成了。其他三种是失败。
>   - task成功执行完，调用`TaskResultGetter.enqueueSuccessfulTask(taskSet, tid, data)`，否则调用`TaskResultGetter.enqueueFailedTask(taskSet, tid, state, data)`。`TaskResultGetter`内部维护了一个线程池，负责异步fetch task执行结果并反序列化。默认开四个线程做这件事，可配参数`"spark.resultGetter.threads"=4`。

QQQQ

**TaskResultGetter取task result的逻辑**

> 1、对于success task，如果taskResult里的数据是直接结果数据，直接把data反序列出来得到结果；如果不是，会调用`blockManager.getRemoteBytes(blockId)`从远程获取。如果远程取回的数据是空的，那么会调用`TaskScheduler.handleFailedTask`，告诉它这个任务是完成了的但是数据是丢失的。否则，取到数据之后会通知`BlockManagerMaster`移除这个block信息，调用`TaskScheduler.handleSuccessfulTask`，告诉它这个任务是执行成功的，并且把result data传回去。
>
> 2、对于failed task，从data里解析出fail的理由，调用`TaskScheduler.handleFailedTask`，告诉它这个任务失败了，理由是什么。

#### 2.6.5 SchedulerBackend

在`TaskScheduler`下层，用于对接不同的资源管理系统，`SchedulerBackend`是个接口，需要实现的主要方法如下：

```
def start(): Unit
def stop(): Unit
def reviveOffers(): Unit // 重要方法：SchedulerBackend把自己手头上的可用资源交给TaskScheduler，TaskScheduler根据调度策略分配给排队的任务吗，返回一批可执行的任务描述，SchedulerBackend负责launchTask，即最终把task塞到了executor模型上，executor里的线程池会执行task的run()
def killTask(taskId: Long, executorId: String, interruptThread: Boolean): Unit =
    throw new UnsupportedOperationException
```

粗粒度：进程常驻的模式，典型代表是standalone模式，mesos粗粒度模式，yarn

细粒度：mesos细粒度模式

维护executor相关信息(包括executor的地址、通信端口、host、总核数，剩余核数)，手头上executor有多少被注册使用了，有多少剩余，总共还有多少核是空的等等。

主要职能

> 1、Driver端主要通过actor监听和处理下面这些事件： 
>
> - - `RegisterExecutor(executorId, hostPort, cores, logUrls)`。这是executor添加的来源，通常worker拉起、重启会触发executor的注册。`CoarseGrainedSchedulerBackend`把这些executor维护起来，更新内部的资源信息，比如总核数增加。最后调用一次`makeOffer()`，即把手头资源丢给`TaskScheduler`去分配一次，返回任务描述回来，把任务launch起来。这个`makeOffer()`的调用会出现在*任何与资源变化相关的事件*中，下面会看到。
>   - `StatusUpdate(executorId, taskId, state, data)`。task的状态回调。首先，调用`TaskScheduler.statusUpdate`上报上去。然后，判断这个task是否执行结束了，结束了的话把executor上的freeCore加回去，调用一次`makeOffer()`。
>   - `ReviveOffers`。这个事件就是别人直接向`SchedulerBackend`请求资源，直接调用`makeOffer()`。
>   - `KillTask(taskId, executorId, interruptThread)`。这个killTask的事件，会被发送给executor的actor，executor会处理`KillTask`这个事件。
>   - `StopExecutors`。通知每一个executor，处理`StopExecutor`事件。
>   - `RemoveExecutor(executorId, reason)`。从维护信息中，那这堆executor涉及的资源数减掉，然后调用`TaskScheduler.executorLost()`方法，通知上层我这边有一批资源不能用了，你处理下吧。`TaskScheduler`会继续把`executorLost`的事件上报给`DAGScheduler`，原因是`DAGScheduler`关心shuffle任务的output location。`DAGScheduler`会告诉`BlockManager`这个executor不可用了，移走它，然后把所有的stage的shuffleOutput信息都遍历一遍，移走这个executor，并且把更新后的shuffleOutput信息注册到`MapOutputTracker`上，最后清理下本地的`CachedLocations`Map。
>
> `2、reviveOffers()`方法的实现。直接调用了`makeOffers()`方法，得到一批可执行的任务描述，调用`launchTasks`。
>
> `3、launchTasks(tasks: Seq[Seq[TaskDescription]])`方法。 
>
> - - 遍历每个task描述，序列化成二进制，然后发送给每个对应的executor这个任务信息 
>     - 如果这个二进制信息太大，超过了9.2M(默认的akkaFrameSize 10M 减去 默认 为akka留空的200K)，会出错，abort整个taskSet，并打印提醒增大akka frame size
>     - 如果二进制数据大小可接受，发送给executor的actor，处理`LaunchTask(serializedTask)`事件。

## 三、Spark的部署模式

![1571627297386](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571627297386.png)

Spark提供了三种集群下的运行模式：

* Standalone 独立集群
* Mesos, apache mesos
* Yarn, hadoop yarn

### 3.1 Spark on Standalone运行过程
Standalone模式是Spark实现的资源调度框架，其主要的节点有Client节点、Master节点和Worker节点。其中Driver既可以运行在Master节点上中，也可以运行在本地Client端。当用spark-shell交互式工具提交Spark的Job时，Driver在Master节点上运行；当使用spark-submit工具提交Job或者在Eclips、IDEA等开发平台上使用”new SparkConf().setMaster(“spark://master:7077”)”方式运行Spark任务时，Driver是运行在本地Client端上的。

运行过程叙述：
```
1. 提交一个任务，任务叫Application
2. 初始化程序的入口SparkContext， 
     2.1 初始化DAG Scheduler
     2.2 初始化Task Scheduler
3. Task Scheduler向master去进行注册并申请资源（CPU Core和Memory）
4. Master根据SparkContext的资源申请要求和Worker心跳周期内报告的信息决定在哪个Worker上分配资源，然后在该Worker上获取资源，然后启动StandaloneExecutorBackend；顺便初始化好了一个线程池
5. StandaloneExecutorBackend向Driver(SparkContext)注册,这样Driver就知道哪些Executor为他进行服务了。到这个时候其实我们的初始化过程基本完成了，我们开始执行transformation的代码，但是代码并不会真正的运行，直到我们遇到一个action操作。生产一个job任务，进行stage的划分
6. SparkContext将Applicaiton代码发送给StandaloneExecutorBackend；并且SparkContext解析Applicaiton代码，构建DAG图，并提交给DAG Scheduler分解成Stage（当碰到Action操作        时，就会催生Job；每个Job中含有1个或多个Stage，Stage一般在获取外部数据和shuffle之前产生）。
7. 将Stage（或者称为TaskSet）提交给Task Scheduler。Task Scheduler负责将Task分配到相应的Worker，最后提交给StandaloneExecutorBackend执行；
8. 对task进行序列化，并根据task的分配算法，分配task
9. 对接收过来的task进行反序列化，把task封装成一个线程
10. 开始执行Task，并向SparkContext报告，直至Task完成。
11. 资源注销
```
运行过程图形说明：

![1571571834484](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571571834484.png)

### 3.2 Spark on YARN运行过程

YARN是一种统一资源管理机制，在其上面可以运行多套计算框架。目前的大数据技术世界，大多数公司除了使用Spark来进行数据计算，由于历史原因或者单方面业务处理的性能考虑而使用着其他的计算框架，比如MapReduce、Storm等计算框架。Spark基于此种情况开发了Spark on YARN的运行模式，由于借助了YARN良好的弹性资源管理机制，不仅部署Application更加方便，而且用户在YARN集群中运行的服务和Application的资源也完全隔离，更具实践应用价值的是YARN可以通过队列的方式，管理同时运行在集群中的多个服务。

Spark on YARN模式根据Driver在集群中的位置分为两种模式：一种是YARN-Client模式，另一种是YARN-Cluster（或称为YARN-Standalone模式）。

#### 3.2.1YARN框架的运行流程

##### 3.2.1.1. YARN概述
　　YARN 是一个资源调度平台，负责为运算程序提供服务器运算资源，相当于一个分布式的操 作系统平台，而 MapReduce 等运算程序则相当于运行于操作系统之上的应用程序。
　　YARN 是 Hadoop2.x 版本中的一个新特性。它的出现其实是为了解决第一代 MapReduce 编程 框架的不足，提高集群环境下的资源利用率，这些资源包括内存，磁盘，网络，IO等。Hadoop2.X 版本中重新设计的这个 YARN 集群，具有更好的扩展性，可用性，可靠性，向后兼容性，以 及能支持除 MapReduce 以外的更多分布式计算程序。

1. YARN并不清楚用户提交的程序的运行机制
2. YARN只提供运算资源的调度（用户向YARN申请资源，YARN就负责分配资源）
3. YRAN中的主管角色叫ResourceManager
4. YARN中具体提供运算资源的角色叫NodeManager
5. 这样一来，YARN 其实就与运行的用户程序完全解耦，就意味着 YARN 上可以运行各种类 型的分布式运算程序（MapReduce 只是其中的一种），比如 MapReduce、Storm 程序，Spark 程序，Tez ……
6. 所以，Spark、Storm 等运算框架都可以整合在 YARN 上运行，只要他们各自的框架中有 符合 YARN 规范的资源请求机制即可
7. yarn 就成为一个通用的资源调度平台，从此，企业中以前存在的各种运算集群都可以整 合在一个物理集群上，提高资源利用率，方便数据共享

##### 3.2.1.2. 优点

　　YARN/MRv2 最基本的想法是将原 JobTracker 主要的资源管理和 Job 调度/监视功能分开作为 两个单独的守护进程。有一个全局的 ResourceManager(RM)和每个 Application 有一个 ApplicationMaster(AM)，Application 相当于 MapReduce Job 或者 DAG jobs。ResourceManager 和 NodeManager(NM)组成了基本的数据计算框架。ResourceManager 协调集群的资源利用， 任何 Client 或者运行着的 applicatitonMaster 想要运行 Job 或者 Task 都得向 RM 申请一定的资 源。ApplicatonMaster 是一个框架特殊的库，对于 MapReduce 框架而言有它自己的 AM 实现， 用户也可以实现自己的 AM，在运行的时候，AM 会与 NM 一起来启动和监视 Tasks。

##### 3.2.1.3. YARN的角色组成

##### 3.2.1.3.1 ResourceManager

　　ResourceManager 是基于应用程序对集群资源的需求进行调度的 YARN 集群主控节点，负责 协调和管理整个集群（所有 NodeManager）的资源，响应用户提交的不同类型应用程序的 解析，调度，监控等工作。ResourceManager 会为每一个 Application 启动一个 MRAppMaster， 并且 MRAppMaster 分散在各个 NodeManager 节点

它主要由两个组件构成：**调度器（Scheduler）和应用程序管理器（ApplicationsManager， ASM）**

YARN 集群的主节点 ResourceManager 的职责：

　　　　1、处理客户端请求

　　　　2、启动或监控 MRAppMaster

　　　　3、监控 NodeManager

　　　　4、资源的分配与调度

##### 3.2.1.3.2 NodeManager

　　NodeManager 是 YARN 集群当中真正资源的提供者，是真正执行应用程序的容器的提供者， 监控应用程序的资源使用情况（CPU，内存，硬盘，网络），并通过心跳向集群资源调度器 ResourceManager 进行汇报以更新自己的健康状态。同时其也会监督 Container 的生命周期 管理，监控每个 Container 的资源使用（内存、CPU 等）情况，追踪节点健康状况，管理日 志和不同应用程序用到的附属服务（auxiliary service）。

　　YARN 集群的从节点 NodeManager 的职责：

　　　　1、管理单个节点上的资源

　　　　2、处理来自 ResourceManager 的命令

　　　　3、处理来自 MRAppMaster 的命令

##### 3.2.1.3.3 MRAppMaster

　　MRAppMaster 对应一个应用程序，职责是：向资源调度器申请执行任务的资源容器，运行 任务，监控整个任务的执行，跟踪整个任务的状态，处理任务失败以异常情况。

##### 3.2.1.3.4 Container

　　Container 容器是一个抽象出来的逻辑资源单位。容器是由 ResourceManager Scheduler 服务 动态分配的资源构成，它包括了该节点上的一定量 CPU，内存，磁盘，网络等信息，MapReduce 程序的所有 Task 都是在一个容器里执行完成的，容器的大小是可以动态调整的。

##### 3.2.1.3.5 ASM

　　应用程序管理器 ASM 负责管理整个系统中所有应用程序，包括应用程序提交、与调度器协 商资源以启动 MRAppMaster、监控 MRAppMaster 运行状态并在失败时重新启动它等。

##### 3.2.1.3.6 Scheduler

　　调度器根据应用程序的资源需求进行资源分配，不参与应用程序具体的执行和监控等工作 资源分配的单位就是 Container，调度器是一个可插拔的组件，用户可以根据自己的需求实 现自己的调度器。YARN 本身为我们提供了多种直接可用的调度器，比如 FIFO，Fair Scheduler 和 Capacity Scheduler 等。

#### 3.2.1.4. YARN 各角色职责

![1571035476003](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571035476003.png)

#### 3.2.1.5. YARN作业执行的流程

YARN 作业执行流程：

1. 用户向 YARN 中提交应用程序，其中包括 MRAppMaster 程序，启动 MRAppMaster 的命令， 用户程序等。
2. ResourceManager 为该程序分配第一个 Container，并与对应的 NodeManager 通讯，要求 它在这个 Container 中启动应用程序 MRAppMaster。
3. MRAppMaster 首先向 ResourceManager 注册，这样用户可以直接通过 ResourceManager 查看应用程序的运行状态，然后将为各个任务申请资源，并监控它的运行状态，直到运行结束，重复 4 到 7 的步骤。
4. MRAppMaster 采用轮询的方式通过 RPC 协议向 ResourceManager 申请和领取资源。
5. 一旦 MRAppMaster 申请到资源后，便与对应的 NodeManager 通讯，要求它启动任务。
6. NodeManager 为任务设置好运行环境（包括环境变量、JAR 包、二进制程序等）后，将 任务启动命令写到一个脚本中，并通过运行该脚本启动任务。
7. 各个任务通过某个 RPC 协议向 MRAppMaster 汇报自己的状态和进度，以让 MRAppMaster 随时掌握各个任务的运行状态，从而可以在任务败的时候重新启动任务。
8. 应用程序运行完成后，MRAppMaster 向 ResourceManager 注销并关闭自己。

![1571035549571](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1571035549571.png)

 #### 3.2.2 YARN-Client

Yarn-Client模式中，Driver在客户端本地运行，这种模式可以使得Spark Application和客户端进行交互，因为Driver在客户端，所以可以通过webUI访问Driver的状态，默认是http://hadoop1:4040访问，而YARN通过http:// hadoop1:8088访问。

YARN-client的工作流程分为以下几个步骤：

文字说明

> 1. Spark Yarn Client向YARN的ResourceManager申请启动Application Master。同时在SparkContent初始化中将创建DAGScheduler和TASKScheduler等，由于我们选择的是Yarn-Client模式，程序会选择YarnClientClusterScheduler和YarnClientSchedulerBackend；
>
> 2. ResourceManager收到请求后，在集群中选择一个NodeManager，为该应用程序分配第一个Container，要求它在这个Container中启动应用程序的ApplicationMaster，与YARN-Cluster区别的是在该ApplicationMaster不运行SparkContext，只与SparkContext进行联系进行资源的分派；
>
> 3. Client中的SparkContext初始化完毕后，与ApplicationMaster建立通讯，向ResourceManager注册，根据任务信息向ResourceManager申请资源（Container）；
>
> 4. 一旦ApplicationMaster申请到资源（也就是Container）后，便与对应的NodeManager通信，要求它在获得的Container中启动启动CoarseGrainedExecutorBackend，CoarseGrainedExecutorBackend启动后会向Client中的SparkContext注册并申请Task；
>
> 5. Client中的SparkContext分配Task给CoarseGrainedExecutorBackend执行，CoarseGrainedExecutorBackend运行Task并向Driver汇报运行的状态和进度，以让Client随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务；
>
> 6. 应用程序运行完成后，Client的SparkContext向ResourceManager申请注销并关闭自己。

图片说明

![1573008493628](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1573008493628.png)

#### 3.2.3　YARN-Cluster

在YARN-Cluster模式中，当用户向YARN中提交一个应用程序后，YARN将分两个阶段运行该应用程序：第一个阶段是把Spark的Driver作为一个ApplicationMaster在YARN集群中先启动；第二个阶段是由ApplicationMaster创建应用程序，然后为它向ResourceManager申请资源，并启动Executor来运行Task，同时监控它的整个运行过程，直到运行完成。

YARN-cluster的工作流程分为以下几个步骤：

文字说明

文字说明

> 1. Spark Yarn Client向YARN中提交应用程序，包括ApplicationMaster程序、启动ApplicationMaster的命令、需要在Executor中运行的程序等；
>
> 2. ResourceManager收到请求后，在集群中选择一个NodeManager，为该应用程序分配第一个Container，要求它在这个Container中启动应用程序的ApplicationMaster，其中ApplicationMaster进行SparkContext等的初始化；
>
> 3. ApplicationMaster向ResourceManager注册，这样用户可以直接通过ResourceManage查看应用程序的运行状态，然后它将采用轮询的方式通过RPC协议为各个任务申请资源，并监控它们的运行状态直到运行结束；
>
> 4. 一旦ApplicationMaster申请到资源（也就是Container）后，便与对应的NodeManager通信，要求它在获得的Container中启动启动CoarseGrainedExecutorBackend，CoarseGrainedExecutorBackend启动后会向ApplicationMaster中的SparkContext注册并申请Task。这一点和Standalone模式一样，只不过SparkContext在Spark Application中初始化时，使用CoarseGrainedSchedulerBackend配合YarnClusterScheduler进行任务的调度，其中YarnClusterScheduler只是对TaskSchedulerImpl的一个简单包装，增加了对Executor的等待逻辑等；
>
> 5. ApplicationMaster中的SparkContext分配Task给CoarseGrainedExecutorBackend执行，CoarseGrainedExecutorBackend运行Task并向ApplicationMaster汇报运行的状态和进度，以让ApplicationMaster随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务；
>
> 6. 应用程序运行完成后，ApplicationMaster向ResourceManager申请注销并关闭自己。

 图片说明

![1573008626876](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1573008626876.png)

#### 3.2.4　YARN-Client 与 YARN-Cluster 区别

理解YARN-Client和YARN-Cluster深层次的区别之前先清楚一个概念：Application Master。在YARN中，每个Application实例都有一个ApplicationMaster进程，它是Application启动的第一个容器。它负责和ResourceManager打交道并请求资源，获取资源之后告诉NodeManager为其启动Container。从深层次的含义讲YARN-Cluster和YARN-Client模式的区别其实就是ApplicationMaster进程的区别。

> 1、YARN-Cluster模式下，Driver运行在AM(Application Master)中，它负责向YARN申请资源，并监督作业的运行状况。当用户提交了作业之后，就可以关掉Client，作业会继续在YARN上运行，因而YARN-Cluster模式不适合运行交互类型的作业；
>
> 2、YARN-Client模式下，Application Master仅仅向YARN请求Executor，Client会和请求的Container通信来调度他们工作，也就是说Client不能离开。

图片说明

yarn-cluster模式下：

<img src="C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1573008693558.png" alt="1573008693558" style="zoom: 80%;" />

yarn-client模式下：

![1573008722566](C:\Users\lv\AppData\Roaming\Typora\typora-user-images\1573008722566.png)