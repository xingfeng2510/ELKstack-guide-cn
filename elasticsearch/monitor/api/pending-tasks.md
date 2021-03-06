# 等待执行的任务列表

首先需要解释的是，这个接口返回的列表，只是对于集群的 master 节点来说的等待执行。数据节点本身的写入和查询线程，如果因为较慢导致排队，是不在这个列表里的。

之前章节已经讲过，master 节点负责处理的任务其实很少，只有集群状态的数据维护。所以绝大多数情况下，这个任务列表应该都是空的。

```
# curl -XGET http://127.0.0.1:9200/_cluster/pending_tasks
{
   "tasks": []
}
```

但是如果你碰上集群有异常，比如频繁有索引映射更新，数据恢复啊，分片分配或者初始化的时候反复出错啊这种时候，就会看到一些任务列表了：

{
  "tasks" : [ {
    "insert_order": 767003,
    "priority": "URGENT",
    "source": "create-index [logstash-2015.06.01], cause [api]",
    "time_in_queue_millis": 86,
    "time_in_queue": "86ms"
  }, {
    "insert_order" : 767004,
    "priority" : "HIGH",
    "source" : "shard-failed ([logstash-2015.05.31][50], node[F3EGSRWGSvWGFlcZ6K9pRA], [P], s[INITIALIZING]), reason [shard failure [failed recovery][IndexShardGatewayRecoveryException[[logstash-2015.05.31][50] failed to fetch index version after copying it over]; nested: CorruptIndexException[[logstash-2015.05.31][50] Preexisting corrupted index [corrupted_nC9t_ramRHqsbQqZO78KVg] caused by: CorruptIndexException[did not read all bytes from file: read 269 vs size 307 (resource: BufferedChecksumIndexInput(NIOFSIndexInput(path=\"/data1/elasticsearch/data/es1003/nodes/0/indices/logstash-2015.05.31/50/index/_16c.si\")))]\norg.apache.lucene.index.CorruptIndexException: did not read all bytes from file: read 269 vs size 307 (resource: BufferedChecksumIndexInput(NIOFSIndexInput(path=\"/data1/elasticsearch/data/es1003/nodes/0/indices/logstash-2015.05.31/50/index/_16c.si\")))\n\tat org.apache.lucene.codecs.CodecUtil.checkFooter(CodecUtil.java:216)\n\tat org.apache.lucene.codecs.lucene46.Lucene46SegmentInfoReader.read(Lucene46SegmentInfoReader.java:73)\n\tat org.apache.lucene.index.SegmentInfos.read(SegmentInfos.java:359)\n\tat org.apache.lucene.index.SegmentInfos$1.doBody(SegmentInfos.java:462)\n\tat org.apache.lucene.index.SegmentInfos$FindSegmentsFile.run(SegmentInfos.java:923)\n\tat org.apache.lucene.index.SegmentInfos$FindSegmentsFile.run(SegmentInfos.java:769)\n\tat org.apache.lucene.index.SegmentInfos.read(SegmentInfos.java:458)\n\tat org.elasticsearch.common.lucene.Lucene.readSegmentInfos(Lucene.java:89)\n\tat org.elasticsearch.index.store.Store.readSegmentsInfo(Store.java:158)\n\tat org.elasticsearch.index.store.Store.readLastCommittedSegmentsInfo(Store.java:148)\n\tat org.elasticsearch.index.engine.InternalEngine.flush(InternalEngine.java:675)\n\tat org.elasticsearch.index.engine.InternalEngine.flush(InternalEngine.java:593)\n\tat org.elasticsearch.index.shard.IndexShard.flush(IndexShard.java:675)\n\tat org.elasticsearch.index.translog.TranslogService$TranslogBasedFlush$1.run(TranslogService.java:203)\n\tat java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)\n\tat java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)\n\tat java.lang.Thread.run(Thread.java:745)\n]; ]]",
    "executing" : true,
    "time_in_queue_millis" : 2813,
    "time_in_queue" : "2.8s"
  }, {
    "insert_order" : 767005,
    "priority" : "HIGH",
    "source" : "refresh-mapping [logstash-2015.06.03][[curl_debuginfo]]",
    "executing" : false,
    "time_in_queue_millis" : 2787,
    "time_in_queue" : "2.7s"
  }, {
    "insert_order" : 767006,
    "priority" : "HIGH",
    "source" : "refresh-mapping [logstash-2015.05.29][[curl_debuginfo]]",
    "executing" : false,
    "time_in_queue_millis" : 448,
    "time_in_queue" : "448ms"
  } ]
}
```

可以看到列表中的任务都有各自的优先级，URGENT 优先于 HIGH。然后是任务在队列中的排队时间，任务的具体内容等。

在上例中，由于磁盘文件损坏，一个分片中某个 segment 的实际内容和长度对不上，导致分片数据恢复无法正常完成，堵塞了后续的索引映射更新操作。这个错误一般来说不太常见，也只能是关闭索引，或者放弃这部分数据。更常见的可能，是集群存储长期数据导致索引映射数据确实大到了 master 节点内存不足以快速处理的地步。

这时候，根据实际情况，可以有以下几种选择：

* 索引就是特别多：给 master 加内存。
* 索引里字段太多：改用 nested object 方式节省字段数量。
* 索引多到内存就是不够了：把一部分数据拆出来另一个集群。
