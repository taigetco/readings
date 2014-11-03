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
 	<li> `BPServiceActor bpServiceToActive`, 关联到active NN, 个、如果没有active NN, 它为null, 也是`bpServices`的成员</li>
    <li> `CopyOnWriteArrayList<BPServiceActor> bpServices` 包含关联到所有NNs的BPServiceActor, active and standy NNs</li>
    <li> `long lastActiveClaimTxId`, 声称ACTIVE NN最近的transaction ID, 当一个声称ACTIVE的NN发送过一个远低于当前`lastActiveClaimTxId`的transaction ID，就可以断定出现脑裂现象， 参考HDFS-2627
  </ul>
  
* `BlockPoolManager`

