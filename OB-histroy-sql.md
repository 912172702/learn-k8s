# OB-histroy-sql

## AuditStatements

- traceId
- tenantName
- userName
- dbName
- planId
- requestTime
- endTime
- elapsedTime
- affectedRows
- returnRows
- totalWaitTimeMacro
- totalWaits
- rpcCount
- hitPlan
- sqlTime 下面几个的和，bar显示
  - netWaitTime
  - queueTime
  - decodeTime
  - getPlanTime
  - executeTime
- dbTime 下面几个的和，bar显示
  - applicationWaitTime
  - concurrencyWaitTime
  - userIoWaitTime
  - scheduleTime
- cacheHit 下面几个的和，bar显示
  - rowCacheHit
  - bloomFilterCacheHit
  - blockCacheHit
  - blockIndexCacheHit
- diskReads
- retryCnt
- storeReadRowCount
  - memstoreReadRowCount
  - ssstoreReadRowCount
- usedWorkerCount
- querySql

## SqlAudit

private String svrIp;
private long svrPort;
private long requestId;
private long sqlExecId;
private String traceId;
private String  sid;
private String clientIp;
private long clientPort;
private long tenantId;
private String tenantName;
private long userId;
private String userName;
private String userClientIp;
private String dbId;
private String dbName;
private String sqlId;
private String querySql;
private long planId;

private long affectedRows;
private long returnRows;
private long partitionCnt;
private long retCode;
private String qcId;
private long dfoId;
private long sqcId;
private long workerId;
private String event;
private String p1text;
private String p1;
private String p2text;
private String p2;
private String p3text;
private String p3;
private long level;
private long waitClassId;
private long waitClassNum;
private String waitClass;

private String state;
private long waitTimeMicro;
private long totalWaitTimeMicro;
private long totalWaits;
private long rpcCount;
private long planType;
private boolean innerSql;
private boolean executorRpc;
private boolean hitPlan;
private long requestTime;
private long elapsedTime;
private long netTime;
private long netWaitTime;
private long queueTime;
private long decodeTime;
private long getPlanTime;
private long executeTime;
private long applicationWaitTime;
private long concurrencyWaitTime;
private long userIoWaitTime;
private long scheduleTime;
private long rowCacheHit;
private long bloomFilterCacheHit;
private long blockCacheHit;
private long blockIndexCacheHit;
private long diskReads;
private long retryCnt;
private boolean tableScan;
private long consistencyLevel;
private long memstoreReadRowCount;
private long ssstoreReadRowCount;
private long requestMemoryUsed;
private long expectedWorkerCount;
private long usedWorkerCount;
private String schedInfo;
private long fuseRowCacheHit;

完成通过sqlAudit和sqlplanmonitor视图查看历史Sql执行情况。
修改了几个由于网络问题前端显示的bug。
遇到的问题