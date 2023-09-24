---
title: Exploring Apache Kafka Internals and Codebase
author: cef
date: 2024-12-07
categories: [Technical Writing, Open Source]
tags: [Apache Kafka, Open Source]
render_with_liquid: false
description: Trying to dig into the Apache Kafka codebase and make some sense of it.
# redirect_to: https://cefboud.com/posts/exploring-kafka-internals/
---

> In case you're curious about how coding agents work, check out [this post](https://cefboud.com/posts/coding-agents-internals-opencode-deepdive/) where I explore an open-source agent internals.

I have been working with Apache Kafka for a couple of years, and during this same period, I also started learning about Open Source and contributing to some projects. Up until recently, though, I hadn't taken the time to intersect these two areas and explore the Kafka codebase. It's interesting how compartmentalized life can be. But intriguing and cool things can emerge when we mix and break down those walls.

One other thought: I consider myself extremely fortunate to be in a field and at a time where world-class systems and code are just a few clicks away, complete with excellent documentation and a commit history that tell the story of their evolution. It truly is a wonderful time to be alive.


## Goal of this post
The goal of this post is not to explore every nook and cranny of the Kafka codebase. I might be foolish, but not so much as to believe this could be accomplished in a single blog post.
The ambition here is to share parts of the code that caught my attention, things I wondered about, investigated, and hopefully understood. These are what I thought were worth sharing. Writing this post is also an excellent way to keep notes and push myself to think more clearly.

## Primer on Apache Kafka {#primer}
For the purpose of our exploration, let's keep the scope limited to the essentials of Kafka. 

Apache Kafka is one of the standard ways of moving large volumes of data in real-time with fault-tolerance and scalability. Messages are replayable and durable. There are three key players in the Kafka game.

A broker is basically a server that accepts Fetch and Produce requests, storing the data in a log, which is replicated across other brokers. Messages are organized into topics, and each topic is divided into partitions. The replication happens at the partition level. Kafka scales because work can be divided between partitions.

Next, we have producers, which are client application, often Java processes, though they can be any application that speaks the Kafka protocol (a TCP protocol). Producers send data to topics on the brokers. Depending on the problem and constraints (mainly latency and throughput), the producer can be configured for optimal performance.

Finally, consumers, similar to producers, are applications (Java or otherwise) that speak the Kafka protocol and want to read data from the server. Essentially, they retrieve data from the distributed, replicated log written by the producers.

## Entrypoint: kafka-server-start.sh and kafka.Kafka
A natural starting point is `kafka-server-start.sh` (the script used to spin up a broker) which fundamentally [invokes](https://github.com/apache/kafka/blob/38e727fe4d7f992534ff797208b9ad16f81c68a6/bin/kafka-server-start.sh#L44) `kafka-run-class.sh` to run [`kafka.Kafka`](https://github.com/apache/kafka/blob/38e727fe4d7f992534ff797208b9ad16f81c68a6/core/src/main/scala/kafka/Kafka.scala#L87) class. 

[`kafka-run-class.sh`](https://github.com/apache/kafka/blob/38e727fe4d7f992534ff797208b9ad16f81c68a6/bin/kafka-run-class.sh#L353-L354), at its core, is nothing other than a wrapper around the `java` command supplemented with all those nice Kafka options.
 ```bash
 exec "$JAVA" $KAFKA_HEAP_OPTS $KAFKA_JVM_PERFORMANCE_OPTS $KAFKA_GC_LOG_OPTS $KAFKA_JMX_OPTS $KAFKA_LOG4J_CMD_OPTS -cp "$CLASSPATH" $KAFKA_OPTS "$@"
 ```

And the entrypoint to the magic powering modern data streaming? The following [`main`](https://github.com/apache/kafka/blob/38e727fe4d7f992534ff797208b9ad16f81c68a6/core/src/main/scala/kafka/Kafka.scala#L87-L88) method situated in `Kafka.scala` i.e. `kafka.Kafka`

```scala
  try {
      val serverProps = getPropsFromArgs(args)
      val server = buildServer(serverProps)

      // ... omitted ....

      // attach shutdown handler to catch terminating signals as well as normal termination
      Exit.addShutdownHook("kafka-shutdown-hook", () => {
        try server.shutdown()
        catch {
          // ... omitted ....
        }
      })

      try server.startup()
      catch {
       // ... omitted ....
      }
      server.awaitShutdown()
    }
    // ... omitted ....
```

That's it. Parse the properties, build the server, register a shutdown hook, and then start up the server.

The first time I looked at this, it felt like peeking behind the curtain. At the end of the day, the whole magic that is Kafka is just a normal JVM program. But a magnificent one.  It's incredible that this astonishing piece of engineering is open source, ready to be explored and experimented with.

And one more fun bit: [`buildServer`](https://github.com/apache/kafka/blob/38e727fe4d7f992534ff797208b9ad16f81c68a6/core/src/main/scala/kafka/Kafka.scala#L70) is defined just above `main`. This where the timeline splits between Zookeeper and KRaft.

```scala
    val config = KafkaConfig.fromProps(props, doLog = false)
    if (config.requiresZookeeper) {
      new KafkaServer(
        config,
        Time.SYSTEM,
        threadNamePrefix = None,
        enableForwarding = enableApiForwarding(config)
      )
    } else {
      new KafkaRaftServer(
        config,
        Time.SYSTEM,
      )
    }
```
How is `config.requiresZookeeper` determined? it is simply a [result of the presence](https://github.com/apache/kafka/blob/38e727fe4d7f992534ff797208b9ad16f81c68a6/core/src/main/scala/kafka/server/KafkaConfig.scala#L337) of the `process.roles` property in the configuration, which is only present in the Kraft installation.

## Zookepeer connection
Kafka has historically relied on Zookeeper for cluster metadata and coordination. This, of course, has changed with the famous KIP-500, which outlined the transition of metadata management into Kafka itself by using Raft (a well-known consensus algorithm designed to manage a replicated log across a distributed system, also used by Kubernetes). This new approach is called KRaft (who doesn't love mac & cheese?).

If you are unfamiliar with Zookeeper, think of it as the place where the Kafka cluster (multiple brokers/servers) stores the shared state of the cluster (e.g., topics, leaders, ACLs, ISR, etc.). It is a remote, filesystem-like entity that stores data. One interesting functionality Zookeeper offers is Watcher callbacks. Whenever the value of the data changes, all subscribed Zookeeper clients (brokers, in this case) are notified of the change. For example, when a new topic is created, all brokers, which are subscribed to the `/brokers/topics` Znode (Zookeeper's equivalent of a directory/file), are alerted to the change in topics and act accordingly.

Why the move? The KIP goes into detail, but the main points are:
1. Zookeeper has its own way of doing things (security, monitoring, API, etc) on top of Kafka's, this results in a operational overhead (I need to manage two distinct components) but also a cognitive one (I need to know about Zookeeper to work with Kafka).
2. The Kafka Controller has to load the full state (topics, partitions, etc) from Zookeeper over the network. Beyond a certain threshold (~200k partitions), this became a scalability bottleneck for Kafka.
3. ~~A love of mac & cheese~~.

Anyway, all that fun aside, it is amazing how simple and elegant the Kafka codebase interacts and leverages Zookeeper. The journey starts in [`initZkClient`](https://github.com/apache/kafka/blob/104fa57933d6831ed3364a26e88fbee2911d27b8/core/src/main/scala/kafka/server/KafkaServer.scala#L233) function inside the `server.startup()` mentioned in the previous section.

```scala
  private def initZkClient(time: Time): Unit = {
    info(s"Connecting to zookeeper on ${config.zkConnect}")
    _zkClient = KafkaZkClient.createZkClient("Kafka server", time, config, zkClientConfig)
    _zkClient.createTopLevelPaths()
  }
```

`KafkaZkClient` is essentially a wrapper around the Zookeeper java client that offers Kafka-specific operations. `CreateTopLevelPaths` ensures all the configuration exist so they can hold Kafka's metadata. [Notably](https://github.com/apache/kafka/blob/104fa57933d6831ed3364a26e88fbee2911d27b8/core/src/main/scala/kafka/zk/ZkData.scala#L1108):

```scala
    BrokerIdsZNode.path, // /brokers/ids
    TopicsZNode.path, // /brokers/topics
    IsrChangeNotificationZNode.path, // /isr_change_notification
```

One simple example of Zookeeper use is [`createTopicWithAssignment`](https://github.com/apache/kafka/blob/104fa57933d6831ed3364a26e88fbee2911d27b8/core/src/main/scala/kafka/zk/AdminZkClient.scala#L101) which is used by the topic creation command. It has the following line:

```scala
zkClient.setOrCreateEntityConfigs(ConfigType.TOPIC, topic, config)
```

which creates the topic Znode with its configuration.

Other data is also stored in Zookeeper and a lot of clever things are implemented. Ultimately, Kafka is just a Zookeeper client that uses its hierarchical filesystem to store metadata such as topics and broker information in Znodes and registers watchers to be notified of changes.

## Networking: SocketServer, Acceptor, Processor, Handler {#network}
A fascinating aspect of the Kafka codebase is how it handles networking. At its core, Kafka is about processing a massive number of Fetch and Produce requests efficiently.

I like to think about it from its basic building blocks. Kafka builds on top of `java.nio.Channels`. Much like goroutines, multiple channels or requests can be handled in a non-blocking manner within a single thread. A sockechannel listens of on a TCP port, multiple channels/requests registered with a selector which polls continuously waiting for connections to be accepted or data to be read.

As explained in the [Primer section](#primer), Kafka has its own TCP protocol that brokers and clients (consumers, produces) use to communicate with each other. A broker can have multiple listeners (PLAINTEXT, SSL, SASL_SSL), each with its own TCP port. This is managed by the [`SockerServer`](https://github.com/apache/kafka/blob/104fa57933d6831ed3364a26e88fbee2911d27b8/core/src/main/scala/kafka/server/KafkaServer.scala#L384) which is instantiated in the `KafkaServer.startup` method.
Part of documentation for the `SocketServer` reads :

```java
 *    - Handles requests from clients and other brokers in the cluster.
 *    - The threading model is
 *      1 Acceptor thread per listener, that handles new connections.
 *      It is possible to configure multiple data-planes by specifying multiple "," separated endpoints for "listeners" in KafkaConfig.
 *      Acceptor has N Processor threads that each have their own selector and read requests from sockets
 *      M Handler threads that handle requests and produce responses back to the processor threads for writing.
```

This sums it up well. Each `Acceptor` thread listens on a socket and accepts new requests. [Here](https://github.com/apache/kafka/blob/104fa57933d6831ed3364a26e88fbee2911d27b8/core/src/main/scala/kafka/network/SocketServer.scala#L723-L736) is the part where the listening starts:

```scala
  val socketAddress = if (Utils.isBlank(host)) {
      new InetSocketAddress(port)
    } else {
      new InetSocketAddress(host, port)
    }
    val serverChannel = socketServer.socketFactory.openServerSocket(
      endPoint.listenerName.value(),
      socketAddress,
      listenBacklogSize, // `socket.listen.backlog.size` property which determines the number of pending connections
      recvBufferSize)   // `socket.receive.buffer.bytes` property which determines the size of SO_RCVBUF (size of the socket's receive buffer)
    info(s"Awaiting socket connections on ${socketAddress.getHostString}:${serverChannel.socket.getLocalPort}.")
```

Each Acceptor thread is paired with [`num.network.threads`](https://github.com/apache/kafka/blob/104fa57933d6831ed3364a26e88fbee2911d27b8/core/src/main/scala/kafka/network/SocketServer.scala#L527) processor thread.

```scala
 override def configure(configs: util.Map[String, _]): Unit = {
    addProcessors(configs.get(SocketServerConfigs.NUM_NETWORK_THREADS_CONFIG).asInstanceOf[Int])
  }
```

The Acceptor thread's `run` method is beautifully concise. It accepts new connections and closes [throttled](https://kafka.apache.org/documentation/#design_quotas) ones:

```scala
  override def run(): Unit = {
    serverChannel.register(nioSelector, SelectionKey.OP_ACCEPT)
    try {
      while (shouldRun.get()) {
        try {
          acceptNewConnections()
          closeThrottledConnections()
        }
        catch {
          // omitted
        }
      }
    } finally {
      closeAll()
    }
  }
```

`acceptNewConnections` *TCP accepts* the connect then assigns it to one the acceptor's Processor threads in a round-robin manner. Each Processor has a [`newConnections`](https://github.com/apache/kafka/blob/104fa57933d6831ed3364a26e88fbee2911d27b8/core/src/main/scala/kafka/network/SocketServer.scala#L931) queue.

```scala
private val newConnections = new ArrayBlockingQueue[SocketChannel](connectionQueueSize)
```
it is an `ArrayBlockingQueue` which is a `java.util.concurrent` thread-safe, FIFO queue. 

The Processor's [`accept`](https://github.com/apache/kafka/blob/104fa57933d6831ed3364a26e88fbee2911d27b8/core/src/main/scala/kafka/network/SocketServer.scala#L1249) method can add a new request from the Acceptor thread if there is enough space in the queue. If all processors' queues are full, we block until a spot clears up.

The Processor registers new connections with its [`Selector`](https://github.com/apache/kafka/blob/104fa57933d6831ed3364a26e88fbee2911d27b8/core/src/main/scala/kafka/network/SocketServer.scala#L953), which is a instance of `org.apache.kafka.common.network.Selector`, a custom Kafka nioSelector to handle non-blocking multi-connection networking (sending and receiving data across multiple requests without blocking). 
Each connection is uniquely identified using a [`ConnectionId`](https://github.com/apache/kafka/blob/ee4264439ddda7bdebcaa845752b824abba14161/clients/src/main/java/org/apache/kafka/common/network/ServerConnectionId.java#L127)

```scala
localHost + ":" + localPort + "-" + remoteHost + ":" + remotePort + "-" + processorId + "-" + connectionIndex
```

The Processor continuously [polls](https://github.com/apache/kafka/blob/ee4264439ddda7bdebcaa845752b824abba14161/core/src/main/scala/kafka/network/SocketServer.scala#L1097) the `Selector` which is waiting for the receive to complete (data sent by the client is ready to be read), then once it is, the Processor's [`processCompletedReceives`](https://github.com/apache/kafka/blob/ee4264439ddda7bdebcaa845752b824abba14161/core/src/main/scala/kafka/network/SocketServer.scala#L1115) *processes* (validates and authenticates) the request. 
The Acceptor and Processors share a reference to `RequestChannel`. It is actually shared with other Acceptor and Processor threads from other listeners. This[ `RequestChannel`](https://github.com/apache/kafka/blob/ee4264439ddda7bdebcaa845752b824abba14161/core/src/main/scala/kafka/network/RequestChannel.scala#L44) object is a central place through which all requests and responses transit. It is actually the way cross-thread settings such as [`queued.max.requests`](https://github.com/apache/kafka/blob/ee4264439ddda7bdebcaa845752b824abba14161/core/src/main/scala/kafka/network/RequestChannel.scala#L340) (max number of requests across all network threads) is enforced. Once the Processor has authenticated and validated it, it [passes it](https://github.com/apache/kafka/blob/ee4264439ddda7bdebcaa845752b824abba14161/core/src/main/scala/kafka/network/SocketServer.scala#L1150) to the `requestChannel`'s queue.

Enter a new component: the Handler. `KafkaRequestHandler` takes over from the Processor, [handling](https://github.com/apache/kafka/blob/ee4264439ddda7bdebcaa845752b824abba14161/core/src/main/scala/kafka/server/KafkaRequestHandler.scala#L158) requests based on their type (e.g., Fetch, Produce). 

A pool of `num.io.threads` handlers is [instantiated](https://github.com/apache/kafka/blob/ee4264439ddda7bdebcaa845752b824abba14161/core/src/main/scala/kafka/server/KafkaServer.scala#L607C43-L607C66) during `KafkaServer.startup`, with each handler having access to the request queue via the `requestChannel` in the SocketServer.

```scala
        dataPlaneRequestHandlerPool = new KafkaRequestHandlerPool(config.brokerId, socketServer.dataPlaneRequestChannel, dataPlaneRequestProcessor, time,
          config.numIoThreads, s"${DataPlaneAcceptor.MetricPrefix}RequestHandlerAvgIdlePercent", DataPlaneAcceptor.ThreadPrefix)
```

Once handled, responses are queued and sent back to the client by the processor.

That's just a glimpse of the happy path for a simple request. There's much more complexity behind the scenes, but I hope this brief explanation provides some insight into the process. Networking is definitely one of the most fascinating parts of the codebase.





------
## Storage: LogManager, UnifiedLog, mmap and Binary search

Another intriguing aspect of the Kafka codebase is how it manages storage and actual data. This is handled by the [`LogManager`](https://github.com/apache/kafka/blob/3cf8745243026b83d97fbc80f2b1fe03f1455701/core/src/main/scala/kafka/log/LogManager.scala#L66) which is responsible for log creation, retrieval, and cleanup. Kafka, much like a database, is primarily concerned with reading from and writing data to files.

Kafka messages are stored in topics. Each topic has one or multiple partitions. And each partition is a directory containing segments, which are the actual files where the data is stored. A replica of a partition is a directory on another broker holding the exact same segments (files).

Here's what partition 0 of `mytopic` might look like:
```
tree /kafka/kafka-logs/mytopic-0
├── 00000000000000000001.index
├── 00000000000000000001.log
├── 00000000000000000001.timeindex
├── 00000000000000000002.index
├── 00000000000000000002.log
├── 00000000000000000002.timeindex


```
* `.log` files that hold the actual data. 
    Running `bin/kafka-dump-log.sh --files /kafka/kafka-logs/mytopic-0/00000000000000000001.log` prints the messages in the log file:
    ```
    baseOffset: 15946643 lastOffset: 15946643 count: 1 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 68 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 0 CreateTime: 1733636569297 size: 1789 magic: 2 compresscodec: lz4 crc: 1648303417 isvalid: true
    baseOffset: 15946644 lastOffset: 15946645 count: 2 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 68 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 1789 CreateTime: 1733636569297 size: 1698 magic: 2 compresscodec: lz4 crc: 4030506336 isvalid: true
    baseOffset: 15946646 lastOffset: 15946647 count: 2 baseSequence: -1 lastSequence: -1 producerId: -1 producerEpoch: -1 partitionLeaderEpoch: 68 isTransactional: false isControl: false deleteHorizonMs: OptionalLong.empty position: 3487 CreateTime: 1733636574447 size: 2876 magic: 2 compresscodec: lz4 crc: 3721890787 isvalid: true
    ```

* `.index` files are a mapping between message offset and file position e.g. message whose offset is 22 start at byte 799.
    Running `bin/kafka-dump-log.sh --files /kafka/kafka-logs/mytopic-0/00000000000000000001.index` prints the offset mapping:
    ```
    offset: 15946649 position: 6363
    offset: 15946652 position: 10563
    offset: 15946657 position: 16767
    ```
* `.timeindex` files map timestamps to offset e.g. timestamp 1733867113 maps to . They allow Kafka to perform time-based queries. 
    Running `bin/kafka-dump-log.sh --files /kafka/kafka-logs/mytopic-0/00000000000000000001.timeindex` prints the time mapping:
    ```
    timestamp: 1733636574447 offset: 15946647
    timestamp: 1733636579430 offset: 15946654
    timestamp: 1733636580266 offset: 15946658
    ```



The `LogManager` is [initialized](https://github.com/apache/kafka/blob/3cf8745243026b83d97fbc80f2b1fe03f1455701/core/src/main/scala/kafka/server/KafkaServer.scala#L318-L329) during `KafkaServer.startup`. It serves as a mechanism to schedule regular log-related tasks that Kafka requires, such as deleting old logs, performing log compaction, and flushing data to disk. This functionality is clearly demonstrated by its [`startupWithConfigOverrides`](https://github.com/apache/kafka/blob/3cf8745243026b83d97fbc80f2b1fe03f1455701/core/src/main/scala/kafka/log/LogManager.scala#L582-L583) method, which is invoked by the LogManager's `startup` method:
 ```scala
    loadLogs(defaultConfig, topicConfigOverrides, isStray) // this could take a while if shutdown was not clean

    /* Schedule the cleanup task to delete old logs */
    if (scheduler != null) {
      info("Starting log cleanup with a period of %d ms.".format(retentionCheckMs))
      scheduler.schedule("kafka-log-retention",
                         () => cleanupLogs(),
                         initialTaskDelayMs,
                         retentionCheckMs)
      info("Starting log flusher with a default period of %d ms.".format(flushCheckMs))
      scheduler.schedule("kafka-log-flusher",
                         () => flushDirtyLogs(),
                         initialTaskDelayMs,
                         flushCheckMs)

      // omitted ...

      scheduler.scheduleOnce("kafka-delete-logs", // will be rescheduled after each delete logs with a dynamic period
                         () => deleteLogs(),
                         initialTaskDelayMs)
    }
    // omitted ...
 ```
We simply schedule schedule `cleanupLogs`,`flushDirtyLogs` and `deleteLogs` at regular intervals, which, as their names suggest, clean, flush, and delete logs.

But before that, we load the different topic partition logs through `loadLogs`, which creates an instance of `UnifiedLog` (UnifiedLog is the class representing a partition log, including both local logs and tiered-storage remote logs) for each partition directory and populates a concurrent, thread-safe hashmap [`currentLogs`](https://github.com/apache/kafka/blob/3cf8745243026b83d97fbc80f2b1fe03f1455701/core/src/main/scala/kafka/log/LogManager.scala#L93) of type `[TopicPartition, UnifiedLog]` that holds all current logs. `UnifiedLog` is used to interact with the actual files. For example, it appends messages to the active segment using its [`append`](https://github1s.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/log/UnifiedLog.scala#L761-L762) method, which calls the [`append`](https://github1s.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/log/UnifiedLog.scala#L905-L906) method of `LocalLog`.

A [`LocalLog`](https://github.com/apache/kafka/blob/3cf8745243026b83d97fbc80f2b1fe03f1455701/storage/src/main/java/org/apache/kafka/storage/internals/log/LocalLog.java#L71-L72), which is part of the `UnifiedLog` (let's leave remote storage for another time), is essentially a map of `<Long, LogSegment>`, specifically a [`ConcurrentNavigableMap`](https://github.com/apache/kafka/blob/3cf8745243026b83d97fbc80f2b1fe03f1455701/storage/src/main/java/org/apache/kafka/storage/internals/log/LogSegments.java#L42-L43). The key is the base offset of the segment, with `base_offset(n+1) > base_offset(n)` for segments `n` and `n+1`. A [Segment](https://github.com/apache/kafka/blob/3cf8745243026b83d97fbc80f2b1fe03f1455701/storage/src/main/java/org/apache/kafka/storage/internals/log/LogSegment.java#L101-L120) is:
```java
 /**
     * Create a LogSegment with the provided parameters.
     *
     * @param log The file records containing log entries
     * @param lazyOffsetIndex The offset index
     * @param lazyTimeIndex The timestamp index
     *...
     * @param baseOffset A lower bound on the offsets in this segment
     * ...
     */
    public LogSegment(FileRecords log,
                      LazyIndex<OffsetIndex> lazyOffsetIndex,
                      LazyIndex<TimeIndex> lazyTimeIndex,
                      // ...
                      long baseOffset,
                      // ...
                      )
```
A log (data) file, offset and time indices and a base offset.



The index files are kept in memory through mmap (a Linux system call to map files into process memory → fast but memory-consuming; index files are small enough since they don't hold the actual data). Segments are represented in the JVM as LogSegment.java, and appending a record to a segment involves adding it to a FileChannel in memory. It is not immediately flushed to disk, which is why Kafka is fast. Appending records is simply adding entries to RAM. 

Data is flushed to disk from memory at a configurable frequency via `log.flush.scheduler.interval.ms` (default is 1 minute). This is  scheduled during the `LogManager`'s startup and implemented in [`flushDirtyLogs`](https://github.com/apache/kafka/blob/3cf8745243026b83d97fbc80f2b1fe03f1455701/core/src/main/scala/kafka/log/LogManager.scala#L634-L639):

```scala
  private def flushDirtyLogs(): Unit = {
    debug("Checking for dirty logs to flush...")

    for ((topicPartition, log) <- currentLogs.toList ++ futureLogs.toList) {
      try {
        val timeSinceLastFlush = time.milliseconds - log.lastFlushTime
        debug(s"Checking if flush is needed on ${topicPartition.topic} flush interval ${log.config.flushMs}" +
              s" last flushed ${log.lastFlushTime} time since last flush: $timeSinceLastFlush")
        if (timeSinceLastFlush >= log.config.flushMs)
          log.flush(false)
      } catch {
        // omitted
      }
    }
  }
```
We loop over all partitions/logs (UnifiedLog) and call `flush` on the `UnifiedLog`, which calls `flush` on the `LocalLog`, which calls `flush` on the `logSegment`, which contains these [three sweet lines](https://github.com/apache/kafka/blob/3cf8745243026b83d97fbc80f2b1fe03f1455701/storage/src/main/java/org/apache/kafka/storage/internals/log/LogSegment.java#L627-L630):

```java
log.flush();
offsetIndex().flush();
timeIndex().flush();
```

Log is backed by a [ChannelFile](https://github.com/apache/kafka/blob/3cf8745243026b83d97fbc80f2b1fe03f1455701/clients/src/main/java/org/apache/kafka/common/record/FileRecords.java#L51) (abstracted through the [`FileRecords`](https://github.com/apache/kafka/blob/3cf8745243026b83d97fbc80f2b1fe03f1455701/storage/src/main/java/org/apache/kafka/storage/internals/log/LogSegment.java#L79) class) and the two indices are backed by a [`MappedByteBuffer`](https://docs.oracle.com/javase/8/docs/api/java/nio/MappedByteBuffer.html#force--) (mmap). The `flush` forces the persistence to local storage.  It's as simple as that.


Another interesting code path is how a Produce request ends up writing data to disk. In the previous [Networking Section](#network), we stopped at the handler, which matches the request API code to a [`PRODUCE` request](https://kafka.apache.org/protocol#The_Messages_Produce), `Key=0`. `Fetch` (Consume) is [`Key=1`](https://kafka.apache.org/protocol#The_Messages_Fetch), for instance. Anyway, the handler dispatches the request with `Key=0` (Produce) to [handleProduceRequest](https://github1s.com/apache/kafka/blob/trunk/core/src/main/scala/kafka/server/KafkaApis.scala#L198-L199), which then calls [`handleProduceAppend`](https://github.com/apache/kafka/blob/3cf8745243026b83d97fbc80f2b1fe03f1455701/core/src/main/scala/kafka/server/KafkaApis.scala#L746), and in turn calls [`appendToLocalLog`](https://github.com/apache/kafka/blob/3cf8745243026b83d97fbc80f2b1fe03f1455701/core/src/main/scala/kafka/server/ReplicaManager.scala#L817). This leads to two other abstractions before reaching [`appendAsLeader`](https://github.com/apache/kafka/blob/3cf8745243026b83d97fbc80f2b1fe03f1455701/core/src/main/scala/kafka/log/UnifiedLog.scala#L714) in the previously visited `UnifiedLog`, which is a simple wrapper around `append`.

One final piece that I think is quite interesting is that the `AbstractIndex` used by `.index` offsets and `.timeindex` timestamp index makes use of [Binary search](https://github.com/apache/kafka/blob/3cf8745243026b83d97fbc80f2b1fe03f1455701/storage/src/main/java/org/apache/kafka/storage/internals/log/AbstractIndex.java#L511-L539) from the MMAPed index when it receives a request for a specific index.


There are indeed many layers, but we can follow the path, and it becomes quite enjoyable when we see how all the Kafka abstractions are built on top of known Java lego blocks such as File Channels and MappedByteBuffer.

If you want to get in touch, you can find me on Twitter/X at [@moncef_abboud](https://x.com/moncef_abboud).  