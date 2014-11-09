## Cassandra

### 类设计

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
