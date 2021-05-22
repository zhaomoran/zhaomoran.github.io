---
title: debezium-connector-mongodb任务重启后无法重新Snapshot
author: Moran
date: 2021-05-22 09:55:00 +0800
categories: [中间件]
tags: [kafka,kafka-connect,debezuim]
mermaid: true
---

> 主要解决新建debezium-connector-mongodb作业时，未完成的多线程snapshot任务重启后无法重新进行snapshot，进而导致的数据丢失问题。主要涉及的技术有：多线程、JAVA8函数式接口

#### 问题发现

---

新建debezium-connector-mongodb 作业（1.2.5.Final）时，**由于kafka-connector distribute部署模式下会进行rebalance**，导致**task进行重启操作**。发现重启后的**未完成的snapshot任务未重新运行，导致snapshot阶段的数据丢失**，[官方文档](https://debezium.io/documentation/reference/1.2/connectors/mongodb.html#mongodb-performing-a-snapshot)表述该过程应为

`This snapshot will continue until it has copied all collections that match the connector’s filters. If the connector is stopped before the tasks' snapshots are completed, upon restart the connector begins the snapshot again.`

* 第一次启动，snapshot被stop且未完成（完成snapshot的表会打印日志`Finished snapshotting {} records for collection '{}'; total duration '{}'`）

  ![image-20201216174449409](https://raw.githubusercontent.com/zhaomoran/markdown/master/img/image-20201216174449409.png)

* 第二次启动，由于找到了offset（`Found existing offset for replica set `），snapshot未执行，直接进行了oplog的读取

  ![image-20201216200809736](https://raw.githubusercontent.com/zhaomoran/markdown/master/img/image-20201216200809736.png)

在使用**单线程进行snapshot**的connector在**snapshot中断重启后会重新进行snapshot**

* 第一次启动，snapshot被stop且未完成，此时任务会报`java.lang.InterruptedException: Interrupted while snapshotting`异常

  ![image-20201216175644749](https://raw.githubusercontent.com/zhaomoran/markdown/master/img/image-20201216175644749.png)

* 第二次启动，未找到了offset，重新进行snapshot

  ![image-20201216175901420](https://raw.githubusercontent.com/zhaomoran/markdown/master/img/image-20201216175901420.png)

  ![image-20201216175945043](https://raw.githubusercontent.com/zhaomoran/markdown/master/img/image-20201216175945043.png)

  ![image-20201216180024414](https://raw.githubusercontent.com/zhaomoran/markdown/master/img/image-20201216180024414.png)



#### 问题分析

---

从debezium-connector-mongodb[官方文档](https://debezium.io/documentation/reference/1.2/connectors/mongodb.html#mongodb-performing-a-snapshot)可以得知**是否进行snapshot在于offset是否被kafka connect存储，存储的offset是否可用**，而通过debug可以发现不管是否完成，kafka connect中均存储了offset，**但未完成的value中 initsync为true, 而多线程下未完成snapshot的情况下initsync消失(异常的地方)**

* snapshot进行时offset为

  ```json
  key：
  ["mimimi_mongo_test_mongo_mrzhao_3479", {
  	"rs": "replSet30017",
  	"server_id": "mimimi_mongo_test_mongo_mrzhao_3479"
  }]
  value：
  {
  	"sec": 1607604049,
  	"ord": 348,
  	"initsync": true,
  	"h": -1998581077279535871,
  	"stxnid": null
  }
  ```

* snapshot完成后offset为

  ```json
  key：
  ["mimimi_mongo_test_mongo_mrzhao_3479",{
  	"rs":"replSet30017",
  	"server_id":"mimimi_mongo_test_mongo_mrzhao_3479"
  }]
  value：
  {
  	"sec":1607604049,
  	"ord":348,
  	"transaction_id":null,
  	"h":-1998581077279535871,
  	"stxnid":null
  }
  ```

通过分析源码可以发现，单线程和多线程处理在被stop时，多线程会吞掉InterruptedException异常，继续剩余流程，而单线程会直接向上抛出InterruptedException异常，后续流程不在执行，详见[代码](https://github.com/debezium/debezium/blob/v1.2.5.Final/debezium-connector-mongodb/src/main/java/io/debezium/connector/mongodb/MongoDbSnapshotChangeEventSource.java#L331)。

* 多线程

  ```mermaid
  graph LR;
    start---thread1---thread1_吞掉[吞掉_InterruptedException]---stopReplicaSetSnapshot;
    start---thread2---thread2_吞掉[吞掉_InterruptedException]---stopReplicaSetSnapshot;
    start---thread3---thread3_吞掉[吞掉_InterruptedException]---stopReplicaSetSnapshot;
    stopReplicaSetSnapshot---completeSnapshot
  ```

* 单线程

    ```mermaid
    graph LR;
      start---thread---抛出_InterruptedException---不再执行stopReplicaSetSnapshot---不再执行completeSnapshot
    ```

而`stopReplicasetSnapshot`是将`Replicaset`从`snapshotmap`中移除，此后再获取的数据`initsync`不在为`true`，见[代码](https://github.com/debezium/debezium/blob/v1.2.5.Final/debezium-connector-mongodb/src/main/java/io/debezium/connector/mongodb/SourceInfo.java#L211)

```java
/**
  * Get the Kafka Connect detail about the source "offset" for the named database, which describes the given position in the
  * database where we have last read. If the database has not yet been seen, this records the starting position
  * for that database. However, if there is a position for the database, the offset representation is returned.
  *
  * @param replicaSetName the name of the replica set name for which the new offset is to be obtained; may not be null
  * @return a copy of the current offset for the database; never null
  */
public Map<String, ?> lastOffset(String replicaSetName) {
    Position existing = positionsByReplicaSetName.get(replicaSetName);
    if (existing == null) {
        existing = INITIAL_POSITION;
    }
    // 调用stopReplicasetSnapshot后，isInitialSyncOngoing为false，则后续的数据中不在存在INITIAL_SYNC
    if (isInitialSyncOngoing(replicaSetName)) {
        return Collect.hashMapOf(TIMESTAMP, Integer.valueOf(existing.getTime()),
                                 ORDER, Integer.valueOf(existing.getInc()),
                                 OPERATION_ID, existing.getOperationId(),
                                 SESSION_TXN_ID, existing.getSessionTxnId(),
                                 INITIAL_SYNC, true);
    }
    Map<String, Object> offset = Collect.hashMapOf(TIMESTAMP, Integer.valueOf(existing.getTime()),
                                                   ORDER, Integer.valueOf(existing.getInc()),
                                                   OPERATION_ID, existing.getOperationId(),
                                                   SESSION_TXN_ID, existing.getSessionTxnId());

    existing.getTxOrder().ifPresent(txOrder -> offset.put(TX_ORD, txOrder));

    return offset;
}
```

`completeSnapshot`是将最后一条数据进行修改（标识snapshot完成）写入到kafka，[代码如下](https://github.com/debezium/debezium/blob/v1.2.5.Final/debezium-core/src/main/java/io/debezium/pipeline/EventDispatcher.java#L402)：

```java
public void completeSnapshot() throws InterruptedException {
    if (bufferedEvent != null) {
        // It is possible that the last snapshotted table was empty
        // this way we ensure that the last event is always marked as last
        // even if it originates form non-last table
        final DataChangeEvent event = bufferedEvent.get();
        final Struct envelope = (Struct) event.getRecord().value();
        if (envelope.schema().field(Envelope.FieldName.SOURCE) != null) {
            final Struct source = envelope.getStruct(Envelope.FieldName.SOURCE);
            final SnapshotRecord snapshot = SnapshotRecord.fromSource(source);
            if (snapshot == SnapshotRecord.TRUE) {
                SnapshotRecord.LAST.toSource(source);
            }
        }
        queue.enqueue(event);
        bufferedEvent = null;
    }
}
```

此处`bufferedEvent.get()`使用的是函数式接口Supplier的实现方式，[代码如下](https://github.com/debezium/debezium/blob/v1.2.5.Final/debezium-core/src/main/java/io/debezium/pipeline/EventDispatcher.java#L389)，此时才会去真实的生成最后一条数据，前面调用了`stopReplicasetSnapshot`，此时再获取offset中的数据不在存在`INITIAL_SYNC`，标识snapshot完成。

```java
bufferedEvent = () -> {
    SourceRecord record = new SourceRecord(
        offsetContext.getPartition(),
        offsetContext.getOffset(),
        topicName, null,
        keySchema, key,
        dataCollectionSchema.getEnvelopeSchema().schema(), value,
        null, headers);
    return changeEventCreator.createDataChangeEvent(record);
};
```

总结问题发生原因如下，多线程下stop任务时，**吞掉了`InterruptedException`导致继续执行了后续的`completeSnapshot`流程，后续流程中最后一条数据的`initsync`被移除，导致再次重启后认定snapshot已完成。**



#### 问题复现

---

1. 创建一个数据量较大的，且拥有多个表的Mongo数据库，以免snapshot过快完成

2. 创建一个多线程snapshot的mongo作业启动后稍待10s，重启task（非重启作业）

3. 重启后的作业将不再进行snapshot，仅进行oplog读取



#### 问题修复

---

**多线程发生`InterruptedException`时虽然吞掉了异常，但会将abort置为true，在所有的线程shutdown后，通过检测abort主动抛出`InterruptedException`，主动结束后续流程。**