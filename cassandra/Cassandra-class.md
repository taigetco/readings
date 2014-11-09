## Cassandra

### config class

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

### db class

* `Keyspace` <br/>
  ```java
  KSMetaData metadata;
  OpOrder writeOrder;
  ConcurrentHashMap<UUID, ColumnFamilyStore> columnFamilyStores; //cfId --> ColumnFamilyStore
  AbstractReplicationStrategy replicationStrategy;
  ```

### service class

* `StorageService` <br/>
  重要参数
  ```java
  TokenMetadata tokenMetadata; //管理 the token/endpoint metadata 信息 */
  ```

### net class

* `MessagingService` , 包含当前节点的token/identifier. token将会在节点之间gossip传递。这个class还维持其他节点负载信息柱状统计图<br/>
  重要参数
  ```java
  EnumMap<Verb, IVerbHandler>(Verb.class) verbHandlers; //通过verb查询messaging handlers
  SimpleCondition listenGate; //
  NonBlockingHashMap<InetAddress, OutboundTcpConnectionPool> connectionManagers;
  ArrayList<SocketThread> socketThreads;
  ExpiringMap<Integer, CallbackInfo> callbacks; //配置参数DatabaseDescriptor.getMinRpcTimeout
  ```
