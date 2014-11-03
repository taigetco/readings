## Hadoop RPC

###线程概览

![overview](HadoopRPCThreads.jpg)

从线程的整体结构可以看到

* Reader 和 Handler 线程可以配置个数的, 以此来处理大量requests.
* CallQueueManager是一个性能瓶颈, 尤其是采用默认`LinkedBlockingQueue` 情况下.
* 对Client的Response默认是在Handler中直接返回, 当在Response过多, 数据量大, 不能一次性处理时, 转移给Responder处理.


