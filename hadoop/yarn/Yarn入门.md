# Yarn入门

### 出现原因

原来的hadoop 1.X版本，资源管理和mapreduce程序耦合在一起，程序升级困难，资源调度功能弱，所以新开发yarn来专门负责资源管理任务。

主要支持了以下功能

- 横向扩展
- 资源管理，与应用程序解耦，方便升级
- 多租户
- 资源管理高利用率，不用再任务运行完再回收资源了,资源回收更灵活
- 本地策略，程序可以再数据所在的机器上面跑
- 高可用
- 可以支撑多种计算模型，不再局限于MapReduce





### RM重启机制

RM开启了保留工作机制后，保存了application运行时数据，RM重启后会接着执行之前的任务 ，Hadoop 2.6.0支持

一般appliction信息存储在zookeeper集群中





### HA模式

要解决的问题：需要解决常见的单点故障问题

Yarn利用了zookeeper上的组件ActiiveStandbyElector来实现自动故障转移

1. **创建锁节点**：Zookeeper上创建一个名为ActiiveStandbyElectorLock的锁节点，所有RM在启动时候，都会竞争去写入这个临时的Lock节点，Zookeeper会保证只有一个RM创建成功，创建成功RM的就是Active状态，失败的转为Standby状态
2. **注册Watch监听**：Standby状态RM向ActiiveStandbyElectorLock节点注册一个节点变更的Watcher监听，利用临时节点的特性（会话结束节点自动消失），可以快速感知到Active节点的变更情况
3. **准备切换**：Active RM故障后，临时节点消失，其他Standby RM都会被Zookeeper服务端的Watcher事件通知，开始竞争写Lock子节点，创建成功的RM变成Active状态



**脑裂问题**

机器有时会出现假死现象（GC时间长，网络中断，CPU高负载），导致ZK节点重新选举之后，该RM恢复正常，还会认为自己是Active状态

可以通过隔离机制来解决这个问题

YARN的Fetching机制，借助ZK节点的ACL权限控制解决这个问题，创建的根ZNode必须携带ZK的ACL信息，防止其他节点来更新，家私后的RM试图更新ZNode节点信息之后，会发现自己没有权限更新，之后把自己切换为Standby状态



### 架构体系

官方出现的分类：

- ResourceManager：全局资源分配
- NodeManager：单台机器资源管理，根据RM命令启动Container
- ApplictionMaster（AM）：具体应用程序内调度分配资源
- Client：负责提交作业
- Container（资源的抽象）：容器，预留资源，资源隔离



**ResourceManager组件**

- Scheduler（调度器）：根据容量队列等条件，将系统资源分给各个正在运行的应用程序
- Application Manager（应用程序管理器）：管理整个系统中所有应用程序，包括应用程序提交，与调度资源器协商启动Application Master，监控Application Master运行状态以在失败时启动它



**NodeMAnager**

- 定期向RM汇报本节点资源使用情况和Container运行状态
- 处理来自AM的Container启动/停止等请求



**ApplictionMaster**

- 与RM协商取得资源（用Container表示）
- 得到的资源分配给内部任务
- 与NM通信以启停任务
- 监控任务状态，并在任务失败后重新申请资源来重启任务

自带两个实现：基于演示的编程模型（不用关心），基于MapReduce的MRAppMaster



**Container**

- Yarn中的资源抽象，封装了某个节点上的多维资源（CPU、内存、磁盘、网络），AM向RM申请资源时，返回的资源就是Container来表示的，yarn会为每个任务分配Container，该任务只能用该Container中描述的资源；Container不同于MRv1中的slot，它是一个动态资源划分单位，根据应用程序需求动态生成的
- 当下Yarn仅支持CPU内存两种资源，底层使用轻量级资源隔离机制Cgroups（linux内核技术）来资源隔离

yarn让特定的资源空闲下来，使用Container描述它



### 通信协议

基于RPC协议

通信一端是Client，另一端是Server，Client总是主动联系Server

所以yarn是基于拉式（pull-based）通信模型





### MR与Yarn交互流程

1. 提交程序，包括ApplicationMaster程序、启动ApplicationMaster命令、用户程序等
2. ResourceManager为该程序分配第一个Container，与对应的Nodemanager通信，要求它在这个Container中启动应用程序的ApplicationMaster
3. ApplicationMaster向ResourceManager注册，为各个任务申请资源，监控运行状态，直到运行结束，重复步骤4-7
4. ApplicationMaster通过RPC协议向ResourceManager申请领取资源，ResourceManager中Scheduler负责调配资源
5. ApplicationMaster申请到资源后，向对应NodeManager通信，要求启动任务
6. NodeManager为任务设置好运行环境（包括JAR、环境变量、二进制程序），然后将启动任务命令写到一个脚本里，通过运行该脚本启动任务
7. 个任务通过RPC协议向ApplicationMaster汇报自己的状态和进度，让ApplicationMaster随时掌握各个任务的状态，从而可以在任务运行失败时候重启该任务，任务运行过程中，用户随时都可以通过RPC协议来向ApplicationMaster查询当前程序运行状态
8. 应用程序完成后，ApplicationMaster向ResourceManager申请注销自己



### JobHistory服务

查看历史任务信息时候，由于任务已经关闭，无法查看历史的任务信息

当yarn重启后，历史任务数据会一个都看不到

针对以上情况，开启JobHistory服务有助于查看历史任务

主要有两个地方要开启

- JobHistoryServer（JHS）：只存储已完成的MR任务作业信息，不存储Spark、Flink作业信息
- 开启日志聚合，把每个Container的日期集中存储起来，而不是分散在各个Nodemanager本地

开启之后，可以查看所有历史任务信息，而且日志都存在同一个节点上



### TimeLineServer

时间轴服务

JobHistory服务只存储MR任务信息

为了查看Spark、Flink信息，Yarn提供了TimeLineServer服务，通用方式存储检索应用程序当前和历史信息



**职责**

- 存储应用程序特定信息，在Container中，可以把特定信息发送到TimeLine服务器存储，同时提供了REST API，用于查询存储的信息，并可以通过应用程序或者框架特定UI展示
- 保存已完成任务常规信息，不止局限于MR任务，JobHistory已经可以看作是TimeLineServer一部分



开启配置

```xml
    <!-- 以下是Timeline相关设置 -->

    <!-- 设置是否开启/使用Yarn Timeline服务 -->
    <!-- 默认值:false -->
    <property>
        <name>yarn.timeline-service.enabled</name>
        <value>true</value>
        <description>
            In the server side it indicates whether timeline service is enabled or not.
            And in the client side, users can enable it to indicate whether client wants
            to use timeline service. If it's enabled in the client side along with
            security, then yarn client tries to fetch the delegation tokens for the
            timeline server.
        </description>
    </property>
    <!-- 设置RM是否发布信息到Timeline服务器 -->
    <!-- 默认值:false -->
    <property>
        <name>yarn.resourcemanager.system-metrics-publisher.enabled</name>
        <value>true</value>
        <description>
            The setting that controls whether yarn system metrics is
            published on the timeline server or not by RM.
        </description>
    </property>
    <!-- 设置是否从Timeline history-service中获取常规信息,如果为否,则是通过RM获取 -->
    <!-- 默认值:false -->
    <property>
        <name>yarn.timeline-service.generic-application-history.enabled</name>
        <value>true</value>
        <description>
            Indicate to clients whether to query generic application data from 
            timeline history-service or not. If not enabled then application 
            data is queried only from Resource Manager. Defaults to false.
        </description>
    </property>
    <!-- leveldb是用于存放Timeline历史记录的数据库,此参数控制leveldb文件存放路径所在 -->
    <!-- 默认值:${hadoop.tmp.dir}/yarn/timeline,其中hadoop.tmp.dir在core-site.xml中设置 -->
    <property>
        <name>yarn.timeline-service.leveldb-timeline-store.path</name>
        <value>${hadoop.tmp.dir}/yarn/timeline</value>
        <description>Store file name for leveldb timeline store.</description>
    </property>
    <!-- 设置leveldb中状态文件存放路径 -->
    <!-- 默认值:${hadoop.tmp.dir}/yarn/timeline -->
    <property>
        <name>yarn.timeline-service.leveldb-state-store.path</name>
        <value>${hadoop.tmp.dir}/yarn/timeline</value>
        <description>Store file name for leveldb state store.</description>
    </property>
    <!-- 设置Timeline Service Web App的主机名,此处将Timeline服务器部署在集群中的hadoop103上 -->
    <!-- 默认值:0.0.0.0 -->
    <property>
        <name>yarn.timeline-service.hostname</name>
        <value>hadoop103</value>
        <description>The hostname of the timeline service web application.</description>
    </property>
    <!-- 设置timeline server rpc service的地址及端口 -->
    <!-- 默认值:${yarn.timeline-service.hostname}:10200 -->
    <property>
        <name>yarn.timeline-service.address</name>
        <value>${yarn.timeline-service.hostname}:10200</value>
        <description>
            This is default address for the timeline server to start the RPC server.
        </description>
    </property>
    <!-- 设置Timeline Service Web App的http地址及端口,由于yarn.http.policy默认值为HTTP_ONLY,
    因此只需要设置http地址即可,不需要设置https -->
    <!-- 默认值:${yarn.timeline-service.hostname}:8188 -->
    <property>
        <name>yarn.timeline-service.webapp.address</name>
        <value>${yarn.timeline-service.hostname}:8188</value>
        <description>The http address of the timeline service web application.</description>
    </property>
    <!-- 设置Timeline服务绑定的IP地址 -->
    <!-- 默认值:空 -->
    <property>
        <name>yarn.timeline-service.bind-host</name>
        <value>192.168.126.103</value>
        <description>
            The actual address the server will bind to. If this optional address is
            set, the RPC and webapp servers will bind to this address and the port specified in
            yarn.timeline-service.address and yarn.timeline-service.webapp.address, respectively.
            This is most useful for making the service listen to all interfaces by setting to
            0.0.0.0.
        </description>
    </property>
    <!-- 启动Timeline数据自动过期清除 -->
    <!-- 默认值:true -->
    <property>
        <name>yarn.timeline-service.ttl-enable</name>
        <value>true</value>
        <description>Enable age off of timeline store data.</description>
    </property>
    <!-- 设置Timeline数据过期时间,单位ms -->
    <!-- 默认值:604800000,即7天 -->
    <property>
        <name>yarn.timeline-service.ttl-ms</name>
        <value>604800000</value>
        <description>Time to live for timeline store data in milliseconds.</description>
    </property>
    <!-- 设置http是否允许CORS(跨域资源共享,Cross-Origin Resource Sharing) -->
    <!-- 默认值:false -->
    <property>
        <name>yarn.timeline-service.http-cross-origin.enabled</name>
        <value>true</value>
        <description>
            Enables cross-origin support (CORS) for web services where cross-origin web 
            response headers are needed. For example, javascript making a web services 
            request to the timeline server. Defaults to false.
        </description>
    </property>

```





### Yarn命令

查看文档：[yarn命令](https://hadoop.apache.org/docs/r3.1.4/hadoop-yarn/hadoop-yarn-site/YarnCommands.html)



### 资源隔离

Yarn现在只能管理CPU和内存两个资源



### 调度器

Yarn支持三种调度器，有ResourceManager中的组件Scheduler负责

- FIFO Scheduler
- Capacity Scheduler
- Fair Scheduler

Apache版本默认使用Capacity Scheduler，CDH版本默认Fair Scheduler



### 查看过期配置

在官方文档对应版本左下角 -> Deprecated Properties



### Yarn应用开发

- ResourceManager：负责接收作业请求，分配资源，任务调度
- NodeManager：负责每台节点的具体具体资源的隔离，资源用container封装
- ApplicationMaster：每个程序内部资源申请，程序监控



### Yarn提交流程

1. client 向 RM 提交应用程序，其中包括启动该应用的 ApplicationMaster 的必须信息，例如 ApplicationMaster 程序、启动 ApplicationMaster 的命令、用户程序等。
2. ResourceManager 启动一个 container 用于运行 ApplicationMaster。 
3. 启动中的 ApplicationMaster 向 ResourceManager 注册自己，启动成功后与 RM 保持心跳。
4. ApplicationMaster 向 ResourceManager 发送请求，申请相应数目的 container。 
5. ResourceManager 返回 ApplicationMaster 的申请的 containers 信息。申请成功的container，由 ApplicationMaster 进行初始化。container 的启动信息初始化后，AM与对应的 NodeManager 通信，要求 NM 启动 container。AM 与 NM 保持心跳，从而对 NM上运行的任务进行监控和管理。
6. container 运行期间，ApplicationMaster 对 container 进行监控。container 通过 RPC协议向对应的 AM 汇报自己的进度和状态等信息。 
7. 应用运行期间，client 直接与 AM 通信获取应用的状态、进度更新等信息。
8. 应用运行结束后，ApplicationMaster 向 ResourceManager 注销自己，并允许属于它的container 被收回。

