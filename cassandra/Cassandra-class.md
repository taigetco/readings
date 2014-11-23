# Cassandra

## 启动Cassandra过程

### DatabaseDescritor初始化

读取配置文件, 存储到Config类， 从Config类中读取配置参数, 来初始化部分配置类

重点谈论下DatabaseDescriptor对SystemKeyspace的初始化
```java
static void applyConfig(Config config) throws ConfigurationException{
    // Hardcoded system keyspaces
    List<KSMetaData> systemKeyspaces = Arrays.asList(KSMetaData.systemKeyspace());
    assert systemKeyspaces.size() == Schema.systemKeyspaceNames.size();
    for (KSMetaData ksmd : systemKeyspaces)
        //把KSMetaData存入keyspaces, CFId(ColumnFamily Id)存入cfIdMap, 方便查找
        Schema.instance.load(ksmd);
}
```

###Load keyspaces 和 UDFs
所有的keyspaces存储在system.schema_keyspaces中,ColumnFamily都存储在system.schema_columnfamilies，UTMetaData存储在system.schema_usertypes, UDFs存储在system.schema_functions，在CassandraDaemon.setup中触发。
```java
void setup(){
    DatabaseDescriptor.loadSchemas();
    Functions.loadUDFFromSchema();
}
/** load keyspace (keyspace) definitions, but do not initialize the keyspace instances. */
public static void loadSchemas(){
    Schema.instance.load(DefsTables.loadFromKeyspace());
    Schema.instance.updateVersion();
}
```
DefsTables类中全是静态方法，在loadFromKeyspace中主要关注

* ColumnFamilyStore创建及查询，查询具体细节参考查询一章

 由于创建ColumnFamilyStore是在Keyspace类中执行，需要先初始化Keyspace, 在对每个ColumnFamily创建相应的ColumnFamilyStore
 
 Keyspace类结构
 ```java
 //包含当前节点的token/identifier. token将会在节点之间gossip传递, 还维持其他节点负载信息柱状统计图 
 KSMetaData metadata;
 OpOrder writeOrder;
 ConcurrentHashMap<UUID, ColumnFamilyStore> columnFamilyStores; //cfId --> ColumnFamilyStore
 AbstractReplicationStrategy replicationStrategy;
 ```
 初始化
 ```java
 private Keyspace(String keyspaceName, boolean loadSSTables){
        metadata = Schema.instance.getKSMetaData(keyspaceName);//从schema取出KSMetaData
        createReplicationStrategy(metadata);//创建ReplicationStrategy
        this.metric = new KeyspaceMetrics(this);
        for (CFMetaData cfm : new ArrayList<CFMetaData>(metadata.cfMetaData().values()))
        {
            initCf(cfm.cfId, cfm.cfName, loadSSTables);
        }
  }
  //在initCF中创建ColumnFamilyStore
  columnFamilyStores.putIfAbsent(cfId, ColumnFamilyStore.createColumnFamilyStore(this, cfName, loadSSTables));
  //在Keyspace.open中对RowCache初始化
  for (ColumnFamilyStore cfs : keyspaceInstance.getColumnFamilyStores())
      cfs.initRowCache();
  ```
* 对结果的组装

```java
public static Collection<KSMetaData> loadFromKeyspace(){
    //执行查询
    List<Row> serializedSchema = SystemKeyspace.serializedSchema(SystemKeyspace.SCHEMA_KEYSPACES_CF);
    List<KSMetaData> keyspaces = new ArrayList<>(serializedSchema.size());
    for (Row row : serializedSchema){
        if (Schema.invalidSchemaRow(row) || Schema.ignoredSchemaRow(row))
            continue;
        //还需要对ColumnFamily, UserType进行查询
        keyspaces.add(KSMetaData.fromSchema(row, serializedColumnFamilies(row.key), serializedUserTypes(row.key)));
    }
    return keyspaces;
}
//在KSMetaData中对Row执行反序列化
public static KSMetaData fromSchema(Row serializedKs, Row serializedCFs, Row serializedUserTypes){
    Map<String, CFMetaData> cfs = deserializeColumnFamilies(serializedCFs);
    UTMetaData userTypes = new UTMetaData(UTMetaData.fromSchema(serializedUserTypes));
    return fromSchema(serializedKs, cfs.values(), userTypes);
}

//KSMataData结构
String name;
Class<? extends AbstractReplicationStrategy> strategyClass;
Map<String, String> strategyOptions;
Map<String, CFMetaData> cfMetaData; // cfName --> CFMetaData
boolean durableWrites; //是否写入Commit log
UTMetaData userTypes;

//CFMetaData结构, 主要参数
UUID cfId; //内部id, 不会暴露给用户
String ksName;
String cfName;
ColumnFamilyType cfType; //standard　or super
CellNameType comparator; // bytes, long, timeuuid, utf8, etc.
```

##Cassandra查询过程
###创建ColumnFamilyStore
* 创建DataTracker

```java
//Memtable类
class Memtable{
  MemtableAllocator allocator;
  //the write barrier for directing writes to this memtable during a switch
  volatile OpOrder.Barrier writeBarrier;
  ColumnFamilyStore cfs;
  AtomicLong liveDataSize = new AtomicLong(0);
  AtomicLong currentOperations = new AtomicLong(0);
  // We index the memtable by RowPosition only for the purpose of being able
  // to select key range using Token.KeyBound. However put() ensures that we
  // actually only store DecoratedKey.
  ConcurrentNavigableMap<RowPosition, AtomicBTreeColumns> rows = new ConcurrentSkipListMap<>();
  CellNameType initialComparator;
  // the last ReplayPosition owned by this Memtable; all ReplayPositions lower are owned by this or an earlier Memtable
  AtomicReference<ReplayPosition> lastReplayPosition;
  // the "first" ReplayPosition owned by this Memtable; 
  // this is inaccurate, and only used as a convenience to prevent CLSM flushing wantonly
  ReplayPosition minReplayPosition = CommitLog.instance.getContext();
  public Memtable(ColumnFamilyStore cfs){
    this.cfs = cfs;
    this.allocator = MEMORY_POOL.newAllocator();
    this.initialComparator = cfs.metadata.comparator;
    this.cfs.scheduleFlush();
  }
}

//View类
/**
 * An immutable structure holding the current memtable, the memtables pending
 * flush, the sstables for a column family, and the sstables that are active
 * in compaction (a subset of the sstables).
 */
public static class View
{
    /**
     * ordinarily a list of size 1, but when preparing to flush will contain both the memtable we will flush
     * and the new replacement memtable, until all outstanding write operations on the old table complete.
     * The last item in the list is always the "current" memtable.
     */
    private final List<Memtable> liveMemtables;
   　/**
　    * contains all memtables that are no longer referenced for writing and are queued for / in the process of being
      * flushed. In chronologically ascending order.
  　  */
      private final List<Memtable> flushingMemtables;
      public final Set<SSTableReader> compacting;
      public final Set<SSTableReader> sstables;
      public final SSTableIntervalTree intervalTree;

      View(List<Memtable> liveMemtables, List<Memtable> flushingMemtables, Set<SSTableReader> sstables, Set<SSTableReader> compacting, SSTableIntervalTree intervalTree){
          this.liveMemtables = liveMemtables;
          this.flushingMemtables = flushingMemtables;
          this.sstables = sstables;
          this.compacting = compacting;
          this.intervalTree = intervalTree;
      }
｝

//DataTracker构造函数
public DataTracker(ColumnFamilyStore cfstore){
    this.cfstore = cfstore;
    this.view = new AtomicReference<>();
    this.init();
}

void init()
{
    view.set(new View(
            ImmutableList.of(new Memtable(cfstore)),
            ImmutableList.<Memtable>of(),
            Collections.<SSTableReader>emptySet(),
            Collections.<SSTableReader>emptySet(),
            SSTableIntervalTree.empty()));
}
```

* 初始化SSTableReader


### config

* `Schema` <br/>
  ```java
  NonBlockingHashMap<String, KSMetaData> keyspaces; //ksName --> KSMataData
  NonBlockingHashMap<String, Keyspace> keyspaceInstances; //ksName --> Keyspace
  ConcurrentBiMap<Pair<String, String>, UUID> cfIdMap; //Pair<ksName, cfName> --> cfId
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

### sstable

* `SStable`, 在SequenceFile之上创建这个类，按顺序存放数据，但排序方式取决于application. 一个单独的索引文件将被维护，包含SSTable　keys和在SSTable中的偏移位置. 在SSTable被打开时，每隔indexInterval的key会被读到内存。每个SSTable 为keys保存一份bloom filter文件.

  ```java
   Descriptor descriptor;
   Set<Component> components;
   CFMetaData metadata;
   IPartitioner partitioner;
   boolean compression;
   DecoratedKey first, last;
  ```

* `SSTableReader extends SSTable`, 通过Keyspace.onStart打开SSTableReaders; 在此之后通过SSTableWriter.renameAndOpen被创建. 不要在存在的SSTable文件上re-call　open()，使用ColumnFamilyStore保存的references来使用.

  ```java
   SegmentedFile ifile, dfile;
   IndexSummary indexSummary;
   IFilter bf;
   StatsMetadata sstableMetadata;
  ```
