## Cassandra

### config

* `Schema` <br/>
  ```java
  NonBlockingHashMap<String, KSMetaData> keyspaces; //ksName --> KSMataData
  NonBlockingHashMap<String, Keyspace> keyspaceInstances; //ksName --> Keyspace
  ConcurrentBiMap<Pair<String, String>, UUID> cfIdMap; //Pair<ksName, cfName> --> cfId
  ```

* `KSMataData` <br/>
  ```java
  String name;
  Class<? extends AbstractReplicationStrategy> strategyClass;
  Map<String, String> strategyOptions;
  Map<String, CFMetaData> cfMetaData; // cfName --> CFMetaData
  boolean durableWrites; //是否写入Commit log
  UTMetaData userTypes;
  ```

* `CFMataData` <br/>
  必需的参数
  ```java
  UUID cfId; //内部id, 不会暴露给用户
  String ksName;
  String cfName;
  ColumnFamilyType cfType; //standard, super
  CellNameType comparator; // bytes, long, timeuuid, utf8, etc.
  ```

### db

* `Keyspace`, 包含当前节点的token/identifier. token将会在节点之间gossip传递。这个class还维持其他节点负载信息柱状统计图<br/>
  ```java
  KSMetaData metadata;
  OpOrder writeOrder;
  ConcurrentHashMap<UUID, ColumnFamilyStore> columnFamilyStores; //cfId --> ColumnFamilyStore
  AbstractReplicationStrategy replicationStrategy;
  ```

* `ColumnFamilyStore`

  ```java
  OpOrder readOrdering; //track accesses to off-heap memtable storage
  Keyspace keyspace;
  String name; //cfName
  CFMetaData metadata;
  IPartitioner partitioner;
  Directories directories;
  SecondaryIndexManager indexManager;
  DataTracker data; //为当前的ColumnFamily 管理Memtables and SSTables
  WrappingCompactionStrategy compactionStrategyWrapper;
  ```
* `DataTracker`

   ```java
   ColumnFamilyStore cfstore;
   Collection<INotificationConsumer> subscribers
   AtomicReference<View> view;
   ```

* `DataTracker.View`, 一个不可变的结构，包含当前的memtable, 等待被flushing的memtables, sstables, 正在compaction的sstables

   ```java
   List<Memtable> liveMemtables; //通常情况，live memtables 都只有一个， 在flushing时， 有两个memtables, 其中一个即将被flushing
   List<Memtable> flushingMemtables; //包含所有不在被写的memtables, 会被排队等待被flush
   Set<SSTableReader> compacting; //是sstables的子集
   Set<SSTableReader> sstables;
   SSTableIntervalTree intervalTree;
   ```

* `Memtable`

  ```java
  MemtableAllocator allocator;
  ColumnFamilyStore cfs;
  ConcurrentSkipListMap<RowPosition, AtomicBTreeColumns> rows; //为了使用Token.KeyBound查询key range, 通过RowPosition来索引memtable. put() 方法只需要存储DecoratedKey
  CellNameType initialComparator = cfs.metadata.comparator;
  AtomicReference<ReplayPosition> lastReplayPosition; //被当前memtable拥有最新的ReplayPosition, 所有低于此ReplayPosition被包含在当前的或者早期的memtable
  ReplayPosition minReplayPosition = CommitLog.instance.getContext(); //Memtable包含的第一个ReplayPosition, 不够精确
  ```

### service

* `StorageService` <br/>
  重要参数<br/>
  
  ```java
  TokenMetadata tokenMetadata; //管理 the token/endpoint metadata 信息 */
  ```

### net

* `MessagingService` <br/>
  重要参数<br/>
  
  ```java
  EnumMap<Verb, IVerbHandler>(Verb.class) verbHandlers; //通过verb查询messaging handlers
  SimpleCondition listenGate; //
  NonBlockingHashMap<InetAddress, OutboundTcpConnectionPool> connectionManagers;
  ArrayList<SocketThread> socketThreads;
  ExpiringMap<Integer, CallbackInfo> callbacks; //配置参数DatabaseDescriptor.getMinRpcTimeout
  ```

### locator
* `AbstractReplicationStrategy` 所有replication strategies的父类 <br/>

  ```java
  String keyspaceName;
  Keyspace keyspace;
  Map<String, String> configOptions;
  TokenMetadata tokenMetadata;
  IEndpointSnitch snitch;
  NonBlockingHashMap<Token, ArrayList<InetAddress>> cachedEndpoints;
  ```
