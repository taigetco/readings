# Hadoop DataNode

DataNode is a class (and program) that stores a set of blocks for a DFS deployment.  A single deployment can have one or many DataNodes. 在HA和Federated环境,每个DataNode会和多个NameNode通信, 而非单个NameNode.  It also communicates with client code and other DataNodes from time to time.

 DataNodes store a series of named blocks.  The DataNode allows client code to read these blocks, or to write new block data.  The DataNode may also, in response to instructions from its NameNode, delete blocks or copy blocks to/from other DataNodes.

 The DataNode maintains just one critical table: **block-> stream of bytes (of BLOCK_SIZE or less)**
 
 This info is stored on a local disk.  The DataNode reports the table's contents to the NameNode upon startup and every so often afterwards.

 DataNodes spend their lives in an endless loop of asking the NameNode for something to do.  A NameNode cannot connect to a DataNode directly; a NameNode simply returns values from functions invoked by a DataNode.

 DataNodes maintain an open server socket so that client code or other DataNodes can read/write data. The host/port for this server is reported to the NameNode, which then sends that information to clients or other DataNodes that might be interested.

##DataNode 初始化

```java
// args传入参数 -rollback 或则　-regular
DataNode datanode = createDataNode(args, null, resources)
```
### 1. 基本流程
```
Collection<StorageLocation> locations = getStorageLocations(conf);
检查dataLocations;
DataNode(conf, locations, resources);
```

### 2. DataNode构造函数过程
```java
volatile boolean shouldRun = true;
volatile boolean shutdownForUpgrade = false;
private boolean shutdownInProgress = false;
private volatile boolean heartbeatsDisabledForTests = false;
private boolean hasAnyBlockPoolRegistered = false;
protected final int checkDiskErrorInterval = 5*1000;

this.lastDiskErrorCheck = 0;
this.maxNumberOfBlocksToLog = conf.getLong(DFS_MAX_NUM_BLOCKS_TO_LOG_KEY,
        DFS_MAX_NUM_BLOCKS_TO_LOG_DEFAULT);
this.usersWithLocalPathAccess = Arrays.asList(
     conf.getTrimmedStrings(DFSConfigKeys.DFS_BLOCK_LOCAL_PATH_ACCESS_USER_KEY));
this.connectToDnViaHostname = conf.getBoolean(
     DFSConfigKeys.DFS_DATANODE_USE_DN_HOSTNAME,
     DFSConfigKeys.DFS_DATANODE_USE_DN_HOSTNAME_DEFAULT);
this.getHdfsBlockLocationsEnabled = conf.getBoolean(
     DFSConfigKeys.DFS_HDFS_BLOCKS_METADATA_ENABLED, 
     DFSConfigKeys.DFS_HDFS_BLOCKS_METADATA_ENABLED_DEFAULT);
this.supergroup = conf.get(DFSConfigKeys.DFS_PERMISSIONS_SUPERUSERGROUP_KEY,
     DFSConfigKeys.DFS_PERMISSIONS_SUPERUSERGROUP_DEFAULT);
this.isPermissionEnabled = conf.getBoolean(
     DFSConfigKeys.DFS_PERMISSIONS_ENABLED_KEY,
     DFSConfigKeys.DFS_PERMISSIONS_ENABLED_DEFAULT);
confVersion = "core-" +
     conf.get("hadoop.common.configuration.version", "UNSPECIFIED") +
     ",hdfs-" +
     conf.get("hadoop.hdfs.configuration.version", "UNSPECIFIED");
// Determine whether we should try to pass file descriptors to clients.
if (conf.getBoolean(DFSConfigKeys.DFS_CLIENT_READ_SHORTCIRCUIT_KEY,
         DFSConfigKeys.DFS_CLIENT_READ_SHORTCIRCUIT_DEFAULT)) {
   String reason = DomainSocket.getLoadingFailureReason();
   if (reason != null) {
       this.fileDescriptorPassingDisabledReason = reason;
   } else {
        this.fileDescriptorPassingDisabledReason = null;
   }
} else {
   this.fileDescriptorPassingDisabledReason =
       "File descriptor passing was not configured.";
}
hostName = getHostName(conf);
startDataNode(conf, dataDirs, resources);
```
    1. 从配置文件初始化配置参数
    2. 执行startDataNode 
   
###3. startDataNode执行流程

```java
this.secureResources = resources;
this.dataDirs = dataDirs;
this.conf = conf;
this.dnConf = new DNConf(conf);//创建DNConf
checkSecureConfig(dnConf, conf, resources);
this.spanReceiverHost = SpanReceiverHost.getInstance(conf);
检查NativeIO, 主要关于Memory Lock 和 Cache;
storage = new DataStorage(); //4. 初始化DataStorage
initDataXceiver(conf);// 5. 初始化DataXceiver
startInfoServer(conf);
pauseMonitor = new JvmPauseMonitor(conf);//参考NameNode
pauseMonitor.start();
this.blockPoolTokenSecretManager = new BlockPoolTokenSecretManager();
initIpcServer(conf);
metrics = DataNodeMetrics.create(conf, getDisplayName());
metrics.getJvmMetrics().setPauseMonitor(pauseMonitor);
blockPoolManager = new BlockPoolManager(this); //6. BlockPoolManager
blockPoolManager.refreshNamenodes(conf); //6. BlockPoolManager
// Create the ReadaheadPool from the DataNode context so we can
// exit without having to explicitly shutdown its thread pool.
readaheadPool = ReadaheadPool.getInstance();
saslClient = new SaslDataTransferClient(dnConf.saslPropsResolver,
dnConf.trustedChannelResolver);
saslServer = new SaslDataTransferServer(dnConf, blockPoolTokenSecretManager);

```

###4. 初始化DataStorage
```java
private final Map<String, BlockPoolSliceStorage> bpStorageMap
      = Collections.synchronizedMap(new HashMap<String, BlockPoolSliceStorage>());
super(NodeType.DATA_NODE);
//开启trash功能的block pool集合
trashEnabledBpids = Collections.newSetFromMap(
    new ConcurrentHashMap<String, Boolean>());
```
###5. 初始化DataXceiver
1. 创建TcpPeerServer
2. 利用TcpPeerServer创建DataXceiverServer
3. 如果支持unix domain socket, 创建DomainPeerServer 和 domain DataXceiverServer

```java
private void initDataXceiver(Configuration conf) throws IOException {
    // find free port or use privileged port provided
    TcpPeerServer tcpPeerServer;
    if (secureResources != null) {
      tcpPeerServer = new TcpPeerServer(secureResources);
    } else {
      tcpPeerServer = new TcpPeerServer(dnConf.socketWriteTimeout,
          DataNode.getStreamingAddr(conf));
    }
    tcpPeerServer.setReceiveBufferSize(HdfsConstants.DEFAULT_DATA_SOCKET_SIZE);
    streamingAddr = tcpPeerServer.getStreamingAddr();
    this.threadGroup = new ThreadGroup("dataXceiverServer");
    xserver = new DataXceiverServer(tcpPeerServer, conf, this);
    this.dataXceiverServer = new Daemon(threadGroup, xserver);
    this.threadGroup.setDaemon(true); // auto destroy when empty

    if (conf.getBoolean(DFSConfigKeys.DFS_CLIENT_READ_SHORTCIRCUIT_KEY,
              DFSConfigKeys.DFS_CLIENT_READ_SHORTCIRCUIT_DEFAULT) ||
        conf.getBoolean(DFSConfigKeys.DFS_CLIENT_DOMAIN_SOCKET_DATA_TRAFFIC,
              DFSConfigKeys.DFS_CLIENT_DOMAIN_SOCKET_DATA_TRAFFIC_DEFAULT)) {
      DomainPeerServer domainPeerServer =
                getDomainPeerServer(conf, streamingAddr.getPort());
      if (domainPeerServer != null) {
        this.localDataXceiverServer = new Daemon(threadGroup,
            new DataXceiverServer(domainPeerServer, conf, this));
      }
    }
    this.shortCircuitRegistry = new ShortCircuitRegistry(conf);
  }
```
### 6. BlockPoolManager

####1. 初始化

```java
private final Map<String, BPOfferService> bpByNameserviceId =
    Maps.newHashMap();
private final Map<String, BPOfferService> bpByBlockPoolId =
    Maps.newHashMap();
private final List<BPOfferService> offerServices =
    Lists.newArrayList();
//对refreshNamenode加锁
private final Object refreshNamenodesLock = new Object();
BlockPoolManager(DataNode dn) {
    this.dn = dn;
}
```
####2. 执行refreshNamenodes
```java
//toAdd 要刷新的name service id
//addrMap 结构 <nsId, <nnId, addr>>
for (String nsToAdd : toAdd) {
   ArrayList<InetSocketAddress> addrs =
      Lists.newArrayList(addrMap.get(nsToAdd).values());
   BPOfferService bpos = createBPOS(addrs);
   bpByNameserviceId.put(nsToAdd, bpos);
   offerServices.add(bpos);
}
startAll();
```

####3. 创建BPOfferService
```java
protected BPOfferService createBPOS(List<InetSocketAddress> nnAddrs) {
    return new BPOfferService(nnAddrs, dn);
}
```
####4. 启动BPOfferService
```java
void startAll(){
  UserGroupInformation.getLoginUser().doAs(
    new PrivilegedExceptionAction<Object>() {
      @Override
      public Object run() throws Exception {
        for (BPOfferService bpos : offerServices) {
          bpos.start();
        }
        return null;
      }
    });
}
```
### 7. BPOfferService解析
####1. 初始化
在DataNode, 对每一个block-pool/namespace创建一个BPOfferService实例对象， 它处理DataNodes和NameNodes之间的heartbeats, 包括active和standby的NameNodes. 它对于每个NameNode，它管理一个BPServiceActor实例. 也维持active NameNodes的状态
```java
//关联到active NN, 如果没有active NN, 它为null, 也是bpServices的成员
private BPServiceActor bpServiceToActive = null;
//包含关联到所有NNs的BPServiceActor, active and standy NNs
private final List<BPServiceActor> bpServices =
    new CopyOnWriteArrayList<BPServiceActor>();
//声称ACTIVE NN最近的transaction ID, 当一个声称ACTIVE的NN发送过一个远低于当前lastActiveClaimTxId 的transaction ID，就可以断定出现脑裂现象， 参考HDFS-2627
private long lastActiveClaimTxId = -1;
private final ReentrantReadWriteLock mReadWriteLock =
    new ReentrantReadWriteLock();
BPOfferService(List<InetSocketAddress> nnAddrs, DataNode dn) {
  this.dn = dn;
  for (InetSocketAddress addr : nnAddrs) {
    this.bpServices.add(new BPServiceActor(addr, this));
  }
}
```
####2. 创建BPServiceActor
`new BPServiceActor(addr, this)` //8. BPServiceActor解析

###8. BPServiceActor解析
和active or standby namenode建立链接的线程, 用来执行: 
 <ul>
   <li> Pre-registration handshake with namenode</li>
   <li> Registration with namenode</li>
   <li> Send periodic heartbeats to the namenode</li>
   <li> Handle commands received from the namenode</li>
 </ul>

BPServiceActor对象的四种状态, 默认CONNECTING

CONNECTING, INIT_FAILED, RUNNING, EXITED, FAILED
####1. 初始化
```java
volatile long lastBlockReport = 0;
volatile long lastDeletedReport = 0;
volatile long lastCacheReport = 0;
private volatile long lastHeartbeat = 0;
boolean resetBlockReportTime = true;
volatile long lastCacheReport = 0;
private volatile RunningState runningState = RunningState.CONNECTING;
//block reports一个小时一次, 在其期间, DN会增量地报告变化到它的block list. 
//这个map, key是block ID, 包含没有发送到NN的block list. 
private final Map<DatanodeStorage, PerStoragePendingIncrementalBR>
   pendingIncrementalBRperStorage = Maps.newHashMap();
// IBR = Incremental Block Report. If this flag is set then an IBR will be
// sent immediately by the actor thread without waiting for the IBR timer
// to elapse.
private volatile boolean sendImmediateIBR = false;
private volatile boolean shouldServiceRun = true;

BPServiceActor(InetSocketAddress nnAddr, BPOfferService bpos) {
  this.bpos = bpos;
  this.dn = bpos.getDataNode();
  this.nnAddr = nnAddr;
  this.dnConf = dn.getDnConf();
}
```


* `BPServiceActor`, 一个和active or standby namenode建立链接的线程, 用来执行: 
 <ul>
   <li> Pre-registration handshake with namenode</li>
   <li> Registration with namenode</li>
   <li> Send periodic heartbeats to the namenode</li>
   <li> Handle commands received from the namenode</li>
 </ul>
 重要参数
 <ul>
   <li> `DatanodeProtocolClientSideTranslatorPB bpNamenode`</li>
 </ul>
 
* `BPOfferService`, 在DataNode, 对每一个block-pool/namespace创建一个BPOfferService实例对象， 它处理DataNodes和NameNodes之间的heartbeats, 包括active和standby的NameNodes. 它对于每个NameNode，它管理一个BPServiceActor实例. 也维持active NameNodes的状态。<br/>
  重要参数 <br/>
  ```java
  BPServiceActor bpServiceToActive; //关联到active NN, 如果没有active NN, 它为null, 也是bpServices的成员
  CopyOnWriteArrayList<BPServiceActor> bpServices; //包含关联到所有NNs的BPServiceActor, active and standy NNs
  long lastActiveClaimTxId; //声称ACTIVE NN最近的transaction ID, 当一个声称ACTIVE的NN发送过一个远低于当前lastActiveClaimTxId 的transaction ID，就可以断定出现脑裂现象， 参考HDFS-2627
  ```
  
* `BlockPoolManager`, 为DN管理BPOfferService对象, Creation, removal, starting, stopping, shutdown on BPOfferService 都必须通过这个类. <br/>
  重要参数<br/>
  ```java
  HashMap<String, BPOfferService> bpByNameserviceId; //nameserviceId 和 bp之间的映射
  HashMap<String, BPOfferService> bpByBlockPoolId; //block pool id 和 bp之间的映射
  HashList<BPOfferService> offerServices;
  ```

* `FsVolumeImpl`, volume用来存储replica. <br/>
  重要参数<br/>
  ```java
  ConcurrentHashMap<String, BlockPoolSlice> bpSlices; //block pool id 映射一个BlockPoolSlice
  DF usage;
  ThreadPoolExecutor cacheExecutor; //每个volume线程池处理block到缓存
  ```
* `FsDatasetAsyncDiskService`, 这个类是多个线程池的容器, 每个线程池对应一个volume, 这样可以容易地安排异步的disk操作. <br/>
  ```java
  HashMap<File, ThreadPoolExecutor> executors; //一个file volume对应一个thread pool
  ```

* `FsDatasetImpl implements FsDatasetSpi<FsVolumeImpl>`, 管理data blocks集<br/>
  重要参数<br/>
  ```java
  FsVolumeList volumes; //包含FsVolumeImpl集
  Map<String, DatanodeStorage> storageMap; //storageUuid(Storage directory identifier) 对应一个DatanodeStorage
  FsDatasetAsyncDiskService asyncDiskService;
  FsDatasetCache cacheManager;
  ReplicaMap volumeMap;
  ```
* `FsDatasetCache`, 为FsDatasetImpl管理缓存, 使用mmap(2) and mlock(2).
* `BlockPoolSliceStorage extends Storage`, 为共享一个block pool id的BlockPoolSlice组管理存储，有几个功能
<ul>
  <li> 格式化一个新的block pool 存储</li>
  <li> 恢复存储状态到一致的状态</li>
  <li> 在升级期间取得block pool的快照</li>
  <li> 回滚一个block pool到之前的快照</li>
  <li> Finalizing block storage by deletion of a snapshot</li>
</ul>

* `DataStorage extends Storage` <br/>
  ```java
  Collections.synchronizedMap(new HashMap<String, BlockPoolSliceStorage>()) bpStorageMap; //block pool id 和 storage之间的映射
  ```

###线程概览

* BPServiceActor thread

　执行流程
 
 
