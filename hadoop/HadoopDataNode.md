## Hadoop DataNode

### 类设计

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

  重要参数
  <ul>
    <li> `BPServiceActor bpServiceToActive`, 关联到active NN, 如果没有active NN, 它为null, 也是`bpServices`的成员</li>
    <li> `CopyOnWriteArrayList<BPServiceActor> bpServices` 包含关联到所有NNs的BPServiceActor, active and standy NNs</li>
    <li> `long lastActiveClaimTxId`, 声称ACTIVE NN最近的transaction ID, 当一个声称ACTIVE的NN发送过一个远低于当前lastActiveClaimTxId 的transaction ID，就可以断定出现脑裂现象， 参考HDFS-2627
  </ul>
  
* `BlockPoolManager`, 为DN管理BPOfferService对象, Creation, removal, starting, stopping, shutdown on BPOfferService 都必须通过这个类. <br/>
  重要参数
  <ul>
    <li> `HashMap<String, BPOfferService> bpByNameserviceId`, nameserviceId 和 bp之间的映射</li>
    <li> `HashMap<String, BPOfferService> bpByBlockPoolId` block pool id 和 bp之间的映射</li>
    <li> `HashList<BPOfferService> offerServices` </li>
  </ul>

* `FsVolumeImpl`, volume用来存储replica. <br/>
  重要参数
  <ul>
    <li> `ConcurrentHashMap<String, BlockPoolSlice> bpSlices`, block pool id 映射一个BlockPoolSlice</li>
    <li> `DF usage` </li>
    <li> `ThreadPoolExecutor cacheExecutor`, 每个volume线程池处理block到缓存</li>
  </ul>
* `FsDatasetAsyncDiskService`, 这个类是多个线程池的容器, 每个线程池对应一个volume, 这样可以容易地安排异步的disk操作. <br/>
   `HashMap<File, ThreadPoolExecutor> executors` 一个file volume对应一个thread pool

* `FsDatasetImpl implements FsDatasetSpi<FsVolumeImpl>`, 管理data blocks集<br/>
  重要参数
  <ul>
    <li> `FsVolumeList volumes`, 包含FsVolumeImpl集</li>
    <li> `Map<String, DatanodeStorage> storageMap`, storageUuid(Storage directory identifier) 对应一个DatanodeStorage</li>
    <li> `FsDatasetAsyncDiskService asyncDiskService` </li>
    <li> `FsDatasetCache cacheManager` </li>
    <li> `ReplicaMap volumeMap` </li>
  </ul>
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
 `Collections.synchronizedMap(new HashMap<String, BlockPoolSliceStorage>()) bpStorageMap`, block pool id 和 storage之间的映射
