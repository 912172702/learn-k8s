# OB SQL Detail Active Report 项目设计文档

## 一、后台部分

为了方便开发，便于后期维护，本系统采用前后端分离的方式开发。先设计好接口和JSON数据，前端利用MockJS模拟后台数据，不用管后台的开发进度。这样可以做到前后端的解耦，避免由于前后端开发进度不同带来的麻烦。

### 1.1. 技术栈

#### 1.1.1. Java Web框架

用Java Web技术开发此项目，用到的主要框架如下：

- `SpringBoot` ：集成了SpringMVC、SpringAspect、log4j等开发框架的平台，可以方便快速的搭建JavaWeb后台开发环境。
- `MyBatis`：一个比Hibernate轻量级的ORM框架。
- `Druid`： 阿里巴巴的数据库连接池产品。
- `Lombok`：一个方便开发者开发的工具，以注解的方式自动生成Getter、Setter、Constructor等方法，以注解的方式注入slf4j日志对象，大量减少了代码量。
- `Swagger2`: 一个自动生成API接口文档并提供接口测试网页的框架，大大减小了前后端联调的难度。
- `JWT`：Java的Token生成框架，支持多种加密算法，用于登录、身份验证、过期校验等。
- `Gson`：一个轻量级Json工具，支持Java反射转换。

#### 1.1.2. OceanBase2.x的SQL相关概念

##### 1.1.2.1. 执行计划

之前写了一篇关于SQL Server执行计划的学习总结，[点击查看](https://yuque.antfin-inc.com/ob/platform/mr25eu)

OceanBase的执行计划类似于SQL Server的执行计划，其中关于什么是算子(Operator), 什么是索引扫描(Index Scan), 什么是表扫描(Table Scan)等等在学习总结中写的非常清楚了。

当OceanBase执行SQL时，会先根据SQL去参数化后的SQL Statement查找计划缓存，查看是否已经有缓存的执行计划，如果有，直接获取执行计划，否则，SQL Optimizer会根据SQL Statement制定执行计划，并将计划缓存。视图gv$PLAN_CACHE_PLAN_EXPLAIIN存放缓存的执行计划，它是租户隔离的。

##### 1.1.2.2 并行执行

OceanBase是一个分布式关系数据库，一张表可以划分为不同分区，分布在相同或者不同的物理机上。

当OceanBase查询操作的时候，会扫描不同机器上的分区。因此，SQL在执行之前，在执行计划中，就必须明确这个SQL是否是一个并行执行计划，如果是，则生成并行执行计划，并由主控实例(Session连接的实例)，经过数据的重分布（重分布为了解决表连接的问题），然后分发这些执行子计划。因此，在OceanBase中，用户发送一句SQL请求，可能在分布式环境中产生很多job，每一个job包含一个执行子计划以及应该被分发的执行实例信息。task是job的实例，job和task的关系类似于类和对象的关系，一个task由一台机器上的一个线程执行。

比如扫描一张10000行的表，表有100各个分区，分布在10台机器上。则主控机器的扫描操作分成不同的task，分发给这10台机器，为了快速，每台机器可能有多个线程(task)去并行执行这个job。

### 1.2. 实时模块相关视图

OceanBase内核部分对实时SQL监控的相关视图还没有开发完成，需要我自己去设计视图。我所设计的视图如下。

#### 1.2.1. gv$SQL_MONITOR视图

上面所说的每个task对应一条gv$SQL_MONITOR记录。该视图的详细结构如下。

字段名 | 字段类型 | 默认值 | 备注
------|-----|-----|-----
SQL_MONITOR_ID | VARCHAR(32) | '' | 唯一ID
SVR_IP | VARCHAR(32) | '' | task执行的实例IP
SVR_PORT | INTEGER | 0 | 服务端实例端口
CLIENT_IP | VARCAHR(32) | '' | 客户端IP
CLIENT_PORT | INTEGER | 0 | 客户端端口号
TRACE_ID | VARCHAR(128) | '' | Query的唯一ID
TENANT_ID | INTEGER | '' | 租户ID
TENANT_NAME | VARCHAR(64) | '' | 租户名
USER_ID | INTEGER | 0 | 用户ID
USERNAME | VARCHAR(64) | '' | 用户名
SQL_STATEMENT | VARCHAR(65536) | '' | 参数化后的SQL
QUERY_SQL | VARCHAR(65536) | '' | 实际请求的SQL
SQL_ID | VARCHAR(32) | '' | SQL的ID
STATUS | VARCAHR(32) | '' | task执行状态
HIT_PLAN | BIT | FALSE | 是否命中执行计划
PLAN_ID | INTEGER | 0 | 执行计划ID
SQL_START_TIME |  BIGINT | 0 | task开始执行时间
SQL_END_TIME | BIGINT | 0 | task结束执行时间
CROSS_INSTANCE | BIT | FALSE | 是否跨实例执行
QC_ID | BITINT | 0 | job调度者ID
DFO_ID | BIGINT | 0 | job的ID
SQC_ID | BIGINT | 0 | task的ID
WORKER_ID | BIGINT | 0 | worker的ID
MAX_DOP | INTEGER | 0 | 算子的最大并行度
MAX_DOP_INSTANCES | INTEGER | 0 | 最大并行度算子的跨实例数
SERVERS_REQUEST | INTEGER | 0 | job请求的执行线程数
SERVERS_ALLOCATED | INTEGER | 0 | job实际分配到的线程数
ERROR | BIT | FALSE | task是否出错
ERROR_NUMBER | VARCHAR(32) | '' | 错误代码 
ERROR_MESSAGE | VARCHAR(256) | '' | 错误信息
ELAPSED_TIME | BITINT | 0 | task已经执行时间
QUEUING_TIME | BITINT | 0 | task在队列中的等待时间
DECODE_TIME | BITINT | 0 | task出队列后的解码时间
GET_PLAN_TIME | BITINT | 0 | task获取执行计划的时间
CPU_TIME | BITINT | 0 | task消耗CPU时间
BUFFER_GETS | BITINT | 0 | task读取缓存次数
DISK_READS | BITINT | 0 | task读取物理磁盘次数
DIRECT_WRITES | BITINT | 0 | task写物理磁盘次数
IO_INTERCONNECT_BYTES | BITINT | 0 | task与存储系统交互的字节数
PHYSICAL_READ_REQUESTS | BITINT | 0 | task读物理磁盘请求数
PHYSICAL_READ_BYTES | BITINT | 0 | task读物理磁盘字节数
PHYSICAL_WRITE_REQUESTS | BITINT | 0 | task写物理磁盘请求书
PHYSICAL_WRITE_BYTES | BITINT | 0 | task写物理磁盘字节数
APPLICATION_WAIT_TIME | BITINT | 0 | task应用程序类事件等待时间
CONCURRENCY_WAIT_TIME | BITINT | 0 | task并发类事件等待时间
USER_IO_WAIT_TIME | BITINT | 0 | task用户IO类事件等待时间
ROW_CACHE_HIT | BITINT | 0 | task行缓存命中数 
BLOOM_FILTER_CACHE_HIT | BITINT | 0 | task BLOOM过滤器缓存命中数
BLOCK_CACHE_HIT | BITINT | 0 | task块缓存命中数
BLOCK_INDEX_CACHE_HIT | BITINT | 0 | task块索引缓存命中数
MEMSTORE_READ_ROW_COUNT | BITINT | 0 | task读memstore次数
SSSTORE_READ_ROW_COUNT | BITINT | 0 | task读ssstore次数

**以上的`status`属性,有六种值**

- QUEUED 正在队列中
- EXECUTING 正在执行
- DONE_ERROR 执行错误结束
- DONE_FIRST_N_ROWS 输出前N行后结束
- DONE_ALL_ROWS 输出所有行后结束
- DONE 执行正常结束

#### 1.2.2. gv$PLAN_CACHE_PLAN_EXPLAIN视图

gv$PLAN_CACHE_PLAN_EXPLAIN视图用来缓存执行计划，每个执行计划一个PLAN_ID，和TENANT_ID一起决定一个执行计划（计划缓存是租户隔离的）。具体字段如下。

字段名 | 字段类型 | 默认值 | 备注
------|-----|-----|-----
TENANT_ID | INTEGER | 0 | 租户ID
OPERATOR_ID | INTEGER | 0 | 算子ID
PARENT_ID | INTEGER | 0 | 父算子ID
OPERATOR_INDEX | INTEGER | 0 | 同一个父算子中，该算子是第几个
IP | VARCHAR(32) | '' | 该计划缓存在哪个IP
PORT | INTEGER | 0 | 端口号
PLAN_ID | INTEGER | 0 | 执行计划ID
OPERATOR_NAME | VARCHAR(128) | '' | 算子名
NAME | VARCHAR(128) | '' | 扫描的表名
ROWS | BIGINT | 0 | 估计输出行数
COST | INTEGER | 0 | 估计花费
PROPERTY | VARCHAR(65536) | '' | 其他属性

#### 1.2.3. gv$SQL_PLAN_MONITOR视图

gv$SQL_PLAN_MONITOR视图中的每一条记录着在每个task中，每个算子的执行情况。具体字段如下。

字段名 | 字段类型 | 默认值 | 备注
------|-----|-----|-----
TRACE_ID | VARCHAR(128) | '' | Query唯一ID
SQL_MONITOR_ID | VARCHAR(32) | '' | SQL_MONITOR唯一ID
IP | VARCHAR(32) | '' | 执行该算子的实例IP
PORT | INTEGER | 0 | 执行该算子实例port
PLAN_ID | INTEGER | 0 | 该算子所属的PLAN的ID
OPERATOR_ID | INTEGER | 0 | 该算子的ID
OPERATOR_NAME | VARCAHR(128) | '' | 该算子的名
STATUS | VARCHAR(32) | '' | 算子执行状态
START_TIME | BIGINT | 0 | 算子开始执行时间
END_TIME | BIGINT | 0 | 算子结束时间
FIRST_CHANGE_TIME | BIGINT | 0 | 算子输出第一行的时间
LAST_CHANGE_TIME | BIGINT | 0 | 算子输出最后一行的时间
STARTS | BIGINT | 0 | 算子被执行的次数
PROGRESS | INTEGER | 0 | 算子执行的百分比(估计值)
DEPTH | INTEGER | 0 | 算子所在计划树的深度
EST_COST | INTEGER | 0 | 算子执行的估计花费
EST_TIME | BIGINT | 0 | 算子估计执行时间
EST_CPU_COST | INTEGER | 0 | 算子估计CPU消耗
EST_IO_COST | INTEGER | 0 | 算子估计IO消耗
EST_TEMP_SPACE | BIGINT | 0 | 算子估计占用临时空间字节数
OUTPUT_ROWS | BIGINT | 0 | 已经输出行数
IO_INTERCONNECT_BYTES | BIGINT | 0 | 算子与存储系统交互的字节数
PYHSICAL_READ_REQUESTS | BIGINT | 0 | 算子物理读请求数
PHYSICAL_READ_BYTES | BIGINT | 0 | 算子物理读的字节数
PHYSICAL_WRITE_REQUESTS | BIGINT | 0 | 算子物理写请求数
PHYSICAL_WRITE_BYTES | BIGINT | 0 | 算子物理写字节数
WORKAREA_MEM | BIGINT | 0 | 工作空间内存大小
WORKARRE_MAX_MEM | BIGINT | 0 | 工作空间最大值
WORKAREA_TEMPSEG | BIGINT | 0 | 使用的临时空间大小
WORKAREA_MAX_TEMPSEG | BIGINT | 0 | 最大使用临时空间大小

**以上的`status`属性,有六种值**

- QUEUED 正在队列中
- EXECUTING 正在执行
- DONE_ERROR 执行错误结束
- DONE_FIRST_N_ROWS 输出前N行后结束
- DONE_ALL_ROWS 输出所有行后结束
- DONE 执行正常结束

### 1.3. 实时模块Entity类设计

```java
public class SqlMonitor {
    private String sqlMonitorId;
    private String svrIp;
    private int svrPort;
    private String clientIp;
    private int clientPort;
    private String traceId;
    private int tenantId;
    private String tenantName;
    private int userId;
    private String username;
    private String sqlStatement;
    private String querySql;
    private String sqlId;
    private String status;
    private boolean hitPlan;
    private int planId;
    private long sqlStartTime;
    private long sqlEndTime;
    private boolean crossInstance;
    private long qcId;
    private long dfoId;
    private long sqcId;
    private long workerId;
    private int maxDop;
    private int maxDopInstances;
    private int serversRequest;
    private int serversAllocated;
    private boolean error;
    private String errorNumber;
    private String errorMessage;
    private long elapsedTime;
    private long queuingTime;
    private long decodeTime;
    private long getPlanTime;
    private long cpuTime;
    private long bufferGets;
    private long diskReads;
    private long directWrites;
    private long ioInterconnectBytes;
    private long physicalReadRequests;
    private long physicalReadBytes;
    private long physicalWriteRequests;
    private long physicalWriteBytes;
    private long applicationWaitTime;
    private long concurrencyWaitTime;
    private long userIoWaitTime;
    private long rowCacheHit;
    private long bloomFilterCacheHit;
    private long blockCacheHit;
    private long blockIndexCacheHit;
    private long memstoreReadRowCount;
    private long ssstoreReadRowCount;
}
```

```java
public class PlanExplain {
    private int tenantId;
    private int operatorId;
    private int parentId;
    private int operatorIndex;
    private String ip;
    private int port;
    private int planId;
    private String operatorName;
    private String name;
    private long rows;
    private int cost;
    private String property;
    private List<SqlPlanMonitor> planMonitors = new ArrayList<>();
}
```

```java
public class SqlPlanMonitor {
    private String traceId;
    private String sqlMonitorId;
    private String ip;
    private Integer port;
    private int planId;
    private int operatorId;
    private String operatorName;
    private String status;
    private long startTime;
    private long endTime;
    private long firstChangeTime;
    private long lastChangeTime;
    private long starts;
    private int progress;
    private int depth;
    private int estCost;
    private long estTime;
    private int estCpuCost;
    private int estIoCost;
    private long estTempSpace;
    private long outputRows;
    private long ioInterconnectBytes;
    private long physicalReadRequests;
    private long physicalReadBytes;
    private long physicalWriteRequests;
    private long physicalWriteBytes;
    private long workareaMem;
    private long workareaMaxMem;
    private long workareaTempseg;
    private long workareaMaxTempseg;
}
```

### 1.4. 实时模块接口设计

#### 1.4.1 登录接口

##### 接口地址

`POST /v1/userAuth/login`

##### 请求参数

FormData

 参数名 | Required | 备注
------ | --------| ------
dbName | Y | 数据库名
ip | Y | 连接的实例ip
password | N | 连接的实例密码
platform | Y | mysql/oracle
port | Y | 连接实例的端口号
tenantName | N | 租户名
username | Y | 用户名

##### Response格式

返回状态码

- 1000 成功
- 2000 数据源初始化失败
- 2002 登陆失败，缺少用户登录信息

例

```java
{
  "code" : 1000,
  "msg" : "成功！",
  "obj" : null
}
```

用户身份识别采用JWT的方式，用户登陆后将用户信息用JWT生成一串TOKEN，以Cookie的方式返回给浏览器，浏览器以后每次请求都带上这份TOKEN，后台先拦截用户的请求，验证TOKEN合法后再调用接口

#### 1.4.2. 分页获取实时Query接口

##### 接口地址

`GET /v1/sqlMonitor/statementInfo/list`

##### 请求参数

FormData

参数名 | Required | 备注
------ | --------| ------
pageNumber | Y | 页面号
pageSize | Y | 页面大小
orderBy | Y | 排序列，默认按照SQL开始时间倒序

##### Response格式

返回状态码

- 1000 成功
- 2001 TOKEM过期，重新登录
- 2003 缺少请求参数

例

```java
{
  "code" : 1000,
  "msg" : "成功",
  "obj" : [
    // 实时Query分页数据
  ]
}
```

#### 1.4.3. 获取单个实时Query详情信息

##### 接口地址

`GET /v1/sqlMonitor/statementInfo/get`

##### 请求参数

FormData

参数名 | Required | 备注
------ | --------| ------
traceId | Y | Query的唯一ID

##### Response格式

返回状态码

- 1000 成功
- 2001 TOKEM过期，重新登录
- 2003 缺少请求参数

例

```java
{
  "code" : 1000,
  "msg" : "成功！",
  "obj" : {
    //单条Query详情信息
  }
}
```

### 1.5. 实时模块VO设计

VO类是与前端交互的类，返回给前端的JSON中通常不直接是Entity，而是将需要的数据抽取成一个类，经处理后返回给前端。

#### 1.5.1. PageInfo

PageInfo是保存一页信息，设计如下

```java
public class PageInfo<T>{
  private int pageSIze;
  private int currPage;
  private int totalPagtes;
  private int totalRows;
  private List<T> data  = new ArrayList<>();
}
```

#### 1.5.2 SqlStatement

SqlStatement将Query的信息整合，一个SqlStatement包含一条Query的汇总信息、每条SQL Monitor的List、这个Query对应的Query Plan。如下。

```java
public class SqlStatement {
    private String traceId;
    private String status;
    private long elapsedTime;
    private int planId;
    private int tenantId;
    private String tenantName;
    private String username;
    private long parallel;
    private long queuingTime;
    private long decodeTime;
    private long getPlanTime;
    private long physicalReadRequests;
    private long physicalWriteRequests;
    private long ioRequests; //new
    private long sqlStartTime;
    private long sqlEndTime;
    private String querySql;
    private boolean hitPlan;//
    private long cpuTime; //
    private long ioTime; //
    private long appTime; //
    private long concurrencyTime ;//
    private long dbTime;
    private long bufferGets; //
    private long diskReads; //
    private long diskWrites;//
    private long diskBytes;//
    private int maxDop; //
    private int maxDopInstances; //
    private int serversRequest;//
    private int serversAllocated;//
    private boolean error;//
    private String errorNumber;//
    private String errorMessage;//
    private long rowCacheHit;//
    private long bloomFilterCacheHit;//
    private long blockCacheHit;//
    private long blockIndexCacheHit;//
    private long cacheHit;//
    private long memstoreReadRowCount;//
    private long ssstoreReadRowCount;//
    private List<SqlMonitor> sqlMonitors = new ArrayList<>();
    private List<PlanExplain> planExplains = new ArrayList<>();
}
```

### 1.6. 动态数据源设计

动态数据源是指，服务端需要根据发送请求的用户，或某个特定识别码，动态的切换数据库的连接源。

#### 1.6.1. 为什么要用动态数据源

OceanBase是面向租户的，各个租户的资源是相互隔离的，每个租户又是有多个用户的，同一个租户的不同用户共享租户资源。

OB SQL Detail Active Report是通过数据库连接信息登录的，登录信息不同，比如租户不同、用户不同、ip不同、port不同等等，系统需要同时容纳数个数据库连接。 当每个用户请求时，要根据用户的识别码动态地将数据源切换到该用户登录时创建的数据源。

因此，必须要设计一个动态的数据源，并且根据用户识别码动态切换数据库连接。

#### 1.6.2. 设计方案

本系统用的是阿里巴巴的开源数据源Druid，我需要在每个用户登录时，为他创建一个DruidDataSource实例。

我使用Spring的TransactionManager管理事物，每当一个事物开始时，TransactionManager会自动选择数据源。

考虑到TransactionManager是通过DataSouce接口选择数据源的，于是我通过Debug查看DataSource的源码。观察在哪里选择数据源。

经过观察，我发现有一个实现DataSource接口的抽象类本身是支持多个数据源的，`AbstractRoutingDataSource`。它用一个Map数据结构存储数据源，key是数据源的ID，value是DataSource实例。当事物开启时，通过 `determineCurrentLookupKey`方法选择一个key，通过`determineTargetDataSource`方法根据当前key选择一个数据源并返回。

于是我想可以继承`AbstractRoutingDataSource`并重写这两个方法以动态地获取数据源。

可是如何保证每个不同的用户请求都能准确的设置数据源呢？

_方案一_  静态全局变量

可以设置一个静态的全局变量，每次一个请求来，就把这个变量设置为用户的识别码，数据源Map的key也是用户的识别码，那用户的每次请求就可以通过这个全局变量来选择属于他的数据源。

问题： 这种方案我要保证并发情况下，当前一个请求设置变量后，获取数据源之前，下一个请求不能重新设置这个变量。于是又要设置一个可重入锁来控制串行执行。这样一来，虽然正确性保证了，但是效率极低，每个请求线程都要串行的获取锁。

_方案二_ ThreadLocal

ThreadLocal的内部是一个Map结构，key是当前线程对象，value是线程对应的数据，每个线程对应一个不同的数据。

Tomcat天然适应这种方式。Tomcat处理请求的方式是一种生产者消费者模式，通过线程池保存处理请求的线程，每一个请求是一个task，分配空闲Thread去处理task。因此，每个请求来临时，可以通过ThreadLocal的set方法将用户的识别码保存到ThreadLocal，与当前线程绑定。这样一来，在这个Request的生命周期内，Thread不变，ThreadLocal中该Thread对应的用户ID也不变。

重写的`determineCurrentLookupKey`方法通过ThreadLocal的get方法获取该线程绑定的用户ID，进而获取数据源。

![动态数据源原理图](https://t1.picb.cc/uploads/2019/06/25/gcRYxF.png)

经过考虑，选择 _方案二_ 为该问题的最终解决方案。

## 二、前端部分

前端采用流行的单页应用开发方式。单页应用即，浏览器只在访问主页时请求一次页面(index.html)和需要的js文件，在之后的所有操作中，都不再请求页面，而是通过组件之间的路由跳转并渲染组件，只通过API接口获取数据，因此可以将这个网站当作一个小客户端应用程序，下载即用。

### 2.1. 技术栈

- `ReactJS` 支持数据双向绑定的JS框架，充分利用组件化编程的思想。
- `Redux` React的State统一管理框架，减少父子组件之间信息传递，让父子组件之间解偶。
- `Ant Design` 蚂蚁金服的前端UI框架，封装了数十样精美常用UI组件，加快前端开发速度。
- `MockJS` 一个API接口模拟框架，拦截JS的的Ajax请求，将请求转发到本地JSON数据，但前端应用本身无感知。做到前后端完全分离开发，减小开发难度。

之前写了一篇React学习总结，[点这里查看](https://yuque.antfin-inc.com/ob/platform/bffxik)

### 2.2. 页面规划

#### 2.2.1. 登录页

![登陆页](https://t1.picb.cc/uploads/2019/06/25/gcxG7y.png)

登录页包括如下表单信息

- `ip` 要登录的实例是ip地址。
- `port` 要登录的实例的端口号。
- `tenantName` 租户名
- `dbName` 数据库名
- `username` 用户名
- `password` 密码

#### 2.2.2. SQL列表页

![SQL列表](https://t1.picb.cc/uploads/2019/06/25/gcxHTj.png)

SQL列表页分页地显示每个SQL的执行情况，这个执行情况是大体的执行情况。能够看到如下条目。

- `Status` SQL的当前的执行状态
- `Trace ID` SQL的唯一ID
- `Elapsed Time` SQL执行的时间
- `Tenant Name` 租户名
- `Username` 用户名
- `Parallel` 并行度
- `Database Time` 数据库时间 ，包括CPU时间、应用程序类事件等待时间、I/O类事件等待时间、并发类事件等待时间。
- `I/O Times` I/O类请求次数，包括读磁盘请求次数和写磁盘请求次数。
- `Cache Hit` 缓存命中数， 包括行缓存命中数、Bloom过滤器缓存命中数、块缓存命中数、块索引缓存命中数。
- `Hit Plan` 是否命中计划缓存
- `SQL Start Time` SQL 开始时间
- `SQL End Time` SQL 结束时间
- `Query Text` 完整的SQL请求

#### 2.2.3. SQL详情页

![详情页](https://t1.picb.cc/uploads/2019/06/25/gcx5Zc.jpg)

SQL详情页分为两部分，上面是SQL Overview，可以看到SQL执行的详情概览。

- `General` SQL执行的概括信息。
- `Wait Statistic` 等待事件的统计，对每个等待时间进行统计汇总。
- `IO Statistic` I/O事件的统计，对每种I/O事件的汇总。

下面一部分是Detail部分，展示了SQL执行的具体执行计划，是一个树形结构，显示每个算子的当前执行状态。具体信息如下。

- `Operation` 算子的名称.
- `Name` 算子正在扫描的表名。
- `Status` 算子当前的执行状态。
- `Estimated Rows` 估计的输出行数。
- `Cost` 一个估计花费的量化值。
- `Time Line` 该算子执行的时间线，从何时开始执行，到何时结束，整个Plan在一个时间线上，绿色方块位置的先后，代表执行顺序的先后。
- `Execute Time` 该算子已经执行的时间。
- `Actual Rows` 算子目前为止输出的行数。
- `Memory(Max)` 算子运行过程中占用的最大内存大小。
- `Temp(Max)` 算子运行过程中占用的最大临时空间大小。
- `Other Properties` 算子的其他属性。
- `IO Requests` 算子的IO请求次数，包括读磁盘请求数，写磁盘请求数。
- `IO Bytes` 算子和磁盘交互的字节数，包括写磁盘的字节数和读磁盘的字节数。

其中 `Time Line`是该算子的执行周期，OB中可能同时有很多task在执行这个算子，Time Line的开始时间是所有task中该算子最早的开始时间，结束时间是所有task中该算子最晚的结束时间。

点击Time Line的bar，可以展示该算子在每个task中执行时间线。如下

![timeline](https://t1.picb.cc/uploads/2019/06/25/gcx9wi.jpg)

### 2.3. 跨域问题解决

跨域问题的出现是由于浏览器的同源策略，即，浏览器发送的API请求要和地址栏里的URL拥有相同的协议、域名、端口号，否则，浏览器认为请求是不安全的，拒绝携带Cookie。由于现在流行的web网站都使用前后端分离的方式开发，前端代码和后端代码可能部署在不同的服务器上，所以在程序员没有手动设置允许跨域前，前后端是无法通信的。

目前跨域问题的通常解决办法是设置代理，常见的反向代理工具是Nginx，将前端代码部署在Nginx服务上，Nginx通过配置url映射，将浏览器的请求转发到服务端。这样，浏览器端可以请求和浏览器地址栏相同的域名，Nginx拦截后，根据配置转发到正确的服务端。“欺骗”了浏览器。

而我的解决方式不是使用用Nginix，而是通过后端手动设置response的参数，来允许响应带回Cookie，前端设置全局请求的withCredentials = true，来允许前端请求携带Cookie。当然这种解决方式是不太安全的，后期也很方便整改。

