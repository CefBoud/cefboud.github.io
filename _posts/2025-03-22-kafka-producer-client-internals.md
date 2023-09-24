---
title: "Behind Sending Millions of Messages Per Second: A Look Under the Hood of Kafka Producer"
author: cef
date: 2025-03-22
categories: [Technical Writing, Open Source]
tags: [Apache Kafka, Open Source]
render_with_liquid: false
description: Exploring the code behind the Kafka Producer client.
---

## Intro
When you work with something for a while, you start to feel like you really know it. But just as you truly get to know someone when you live with them, you truly understand a piece of technology when you examine its code. With that in mind, let's take a closer look at the Kafka producer and see if it turns out to be a messy roommate.

## Post Goals
The canonical Kafka Producer looks as follows:

```java
 Properties props = new Properties();
 props.put("bootstrap.servers", "localhost:9092");
 props.put("linger.ms", 1);
 props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
 props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

 Producer<String, String> producer = new KafkaProducer<>(props);
 for (int i = 0; i < 100; i++)
     producer.send(new ProducerRecord<String, String>("my-topic", Integer.toString(i), Integer.toString(i)));

 producer.close();
```

Some properties, a constructor, and a simple send method. This short snippet powers workloads handling millions of messages per second. It's quite impressive.  

One goal is to examine the code behind this code to get a feel for it and demystify its workings. Another is to understand where properties like `batch.size`, `linger.ms`, `acks`, `buffer.memory`, and others fit in, how they balance latency and throughput to achieve the desired performance.


## The Entrypoint: KafkaProducer class
The entrypoint to the Kafka producer is unsurprisingly the `KafkaProducer` class. To keep things simple, we're going to ignore all telemetry and transaction-related code.


###  The Constructor
Let's take a look at the [constructor](https://github.com/apache/kafka/blob/5d2bfb4151700000fc5ec22918ecd3cecb5178a5/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java#L334-L469) (abridged):

```java
    KafkaProducer(ProducerConfig config,
                  Serializer<K> keySerializer,
                  Serializer<V> valueSerializer,
                  ProducerMetadata metadata,
                  KafkaClient kafkaClient,
                  ProducerInterceptors<K, V> interceptors,
                  ApiVersions apiVersions,
                  Time time) {
        try {
            this.producerConfig = config;
            this.time = time;

            this.clientId = config.getString(ProducerConfig.CLIENT_ID_CONFIG);

            LogContext logContext;
            if (transactionalId == null)
                logContext = new LogContext(String.format("[Producer clientId=%s] ", clientId));
            else
                logContext = new LogContext(String.format("[Producer clientId=%s, transactionalId=%s] ", clientId, transactionalId));
            log = logContext.logger(KafkaProducer.class);
            log.trace("Starting the Kafka producer");

            this.partitionerPlugin = Plugin.wrapInstance(
                    config.getConfiguredInstance(
                        ProducerConfig.PARTITIONER_CLASS_CONFIG,
                        Partitioner.class,
                        Collections.singletonMap(ProducerConfig.CLIENT_ID_CONFIG, clientId)),
                    metrics,
                    ProducerConfig.PARTITIONER_CLASS_CONFIG);
            this.partitionerIgnoreKeys = config.getBoolean(ProducerConfig.PARTITIONER_IGNORE_KEYS_CONFIG);
            long retryBackoffMs = config.getLong(ProducerConfig.RETRY_BACKOFF_MS_CONFIG);
            long retryBackoffMaxMs = config.getLong(ProducerConfig.RETRY_BACKOFF_MAX_MS_CONFIG);
            if (keySerializer == null) {
                keySerializer = config.getConfiguredInstance(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, Serializer.class);
                keySerializer.configure(config.originals(Collections.singletonMap(ProducerConfig.CLIENT_ID_CONFIG, clientId)), true);
            } else {
                config.ignore(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG);
            }
            this.keySerializerPlugin = Plugin.wrapInstance(keySerializer, metrics, ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG);

            if (valueSerializer == null) {
                valueSerializer = config.getConfiguredInstance(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, Serializer.class);
                valueSerializer.configure(config.originals(Collections.singletonMap(ProducerConfig.CLIENT_ID_CONFIG, clientId)), false);
            } else {
                config.ignore(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG);
            }
            this.valueSerializerPlugin = Plugin.wrapInstance(valueSerializer, metrics, ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG);


            List<ProducerInterceptor<K, V>> interceptorList = (List<ProducerInterceptor<K, V>>) ClientUtils.configuredInterceptors(config,
                    ProducerConfig.INTERCEPTOR_CLASSES_CONFIG,
                    ProducerInterceptor.class);
            if (interceptors != null)
                this.interceptors = interceptors;
            else
                this.interceptors = new ProducerInterceptors<>(interceptorList, metrics);
            ClusterResourceListeners clusterResourceListeners = ClientUtils.configureClusterResourceListeners(
                    interceptorList,
                    reporters,
                    Arrays.asList(this.keySerializerPlugin.get(), this.valueSerializerPlugin.get()));
            this.maxRequestSize = config.getInt(ProducerConfig.MAX_REQUEST_SIZE_CONFIG);
            this.totalMemorySize = config.getLong(ProducerConfig.BUFFER_MEMORY_CONFIG);
            this.compression = configureCompression(config);

            this.maxBlockTimeMs = config.getLong(ProducerConfig.MAX_BLOCK_MS_CONFIG);
            int deliveryTimeoutMs = configureDeliveryTimeout(config, log);

            this.apiVersions = apiVersions;

            // There is no need to do work required for adaptive partitioning, if we use a custom partitioner.
            boolean enableAdaptivePartitioning = partitionerPlugin.get() == null &&
                config.getBoolean(ProducerConfig.PARTITIONER_ADPATIVE_PARTITIONING_ENABLE_CONFIG);
            RecordAccumulator.PartitionerConfig partitionerConfig = new RecordAccumulator.PartitionerConfig(
                enableAdaptivePartitioning,
                config.getLong(ProducerConfig.PARTITIONER_AVAILABILITY_TIMEOUT_MS_CONFIG)
            );
            // As per Kafka producer configuration documentation batch.size may be set to 0 to explicitly disable
            // batching which in practice actually means using a batch size of 1.
            int batchSize = Math.max(1, config.getInt(ProducerConfig.BATCH_SIZE_CONFIG));
            this.accumulator = new RecordAccumulator(logContext,
                    batchSize,
                    compression,
                    lingerMs(config),
                    retryBackoffMs,
                    retryBackoffMaxMs,
                    deliveryTimeoutMs,
                    partitionerConfig,
                    metrics,
                    PRODUCER_METRIC_GROUP_NAME,
                    time,
                    apiVersions,
                    transactionManager,
                    new BufferPool(this.totalMemorySize, batchSize, metrics, time, PRODUCER_METRIC_GROUP_NAME));

            List<InetSocketAddress> addresses = ClientUtils.parseAndValidateAddresses(config);
            if (metadata != null) {
                this.metadata = metadata;
            } else {
                this.metadata = new ProducerMetadata(retryBackoffMs,
                        retryBackoffMaxMs,
                        config.getLong(ProducerConfig.METADATA_MAX_AGE_CONFIG),
                        config.getLong(ProducerConfig.METADATA_MAX_IDLE_CONFIG),
                        logContext,
                        clusterResourceListeners,
                        Time.SYSTEM);
                this.metadata.bootstrap(addresses);
            }

            this.sender = newSender(logContext, kafkaClient, this.metadata);
            String ioThreadName = NETWORK_THREAD_PREFIX + " | " + clientId;
            this.ioThread = new KafkaThread(ioThreadName, this.sender, true);
            this.ioThread.start();
            config.logUnused();
            AppInfoParser.registerAppInfo(JMX_PREFIX, clientId, metrics, time.milliseconds());
            log.debug("Kafka producer started");
        } catch (Throwable t) {
            // call close methods if internal objects are already constructed this is to prevent resource leak. see KAFKA-2121
            close(Duration.ofMillis(0), true);
            // now propagate the exception
            throw new KafkaException("Failed to construct kafka producer", t);
        }
    }

```
There's a flurry of interesting things happening here. First, let's take note of some producer properties being fetched from the configuration.  

My eyes immediately scan for `BATCH_SIZE_CONFIG`, `lingerMs`, `BUFFER_MEMORY_CONFIG`, and `MAX_BLOCK_MS_CONFIG`.  

We can see `CLIENT_ID_CONFIG` (`client.id`), along with retry-related properties like `RETRY_BACKOFF_MS_CONFIG` and `RETRY_BACKOFF_MAX_MS_CONFIG`.  

The constructor also attempts to dynamically load `PARTITIONER_CLASS_CONFIG`, which specifies a custom partitioner class. Right after that, there's `PARTITIONER_IGNORE_KEYS_CONFIG`, indicating whether key hashes should be used to select a partition in the `DefaultPartitioner` (when no custom partitioner is provided).  

Of course, we also see the Key and Value serializer plugins being initialized. Our Java object-to-bytes translators.  

Two other objects are initialized, which I believe are the real workhorses:  

- `this.accumulator` (`RecordAccumulator`): Holds and accumulates the queues containing record batches.  
- `this.sender` (`Sender`): The thread that iterates over the accumulated batches and sends the ready ones over the network.


We also spot this line which validates the bootstrap servers:

```java
List<InetSocketAddress> addresses = ClientUtils.parseAndValidateAddresses(config);
```
Simplified, it looks as follows:
```java
List<String> urls = config.getList(CommonClientConfigs.BOOTSTRAP_SERVERS_CONFIG);
String clientDnsLookupConfig = config.getString(CommonClientConfigs.CLIENT_DNS_LOOKUP_CONFIG);
        List<InetSocketAddress> addresses = new ArrayList<>();
        for (String url : urls) {
            if (url != null && !url.isEmpty()) {
                    String host = getHost(url);
                    Integer port = getPort(url);
                    if (clientDnsLookup == ClientDnsLookup.RESOLVE_CANONICAL_BOOTSTRAP_SERVERS_ONLY) {
                        InetAddress[] inetAddresses = InetAddress.getAllByName(host);
                        for (InetAddress inetAddress : inetAddresses) {
                            String resolvedCanonicalName = inetAddress.getCanonicalHostName();
                            InetSocketAddress address = new InetSocketAddress(resolvedCanonicalName, port);
                            if (address.isUnresolved()) {
                                log.warn("Couldn't resolve server {} from {} as DNS resolution of the canonical hostname {} failed for {}", url, CommonClientConfigs.BOOTSTRAP_SERVERS_CONFIG, resolvedCanonicalName, host);
                            } else {
                                addresses.add(address);
                            }
                        }
                    } else {
                        InetSocketAddress address = new InetSocketAddress(host, port);
                        if (address.isUnresolved()) {
                            log.warn("Couldn't resolve server {} from {} as DNS resolution failed for {}", url, CommonClientConfigs.BOOTSTRAP_SERVERS_CONFIG, host);
                        } else {
                            addresses.add(address);
                        }
                    }
            }
        }
```

The key objective behind `RESOLVE_CANONICAL_BOOTSTRAP_SERVERS_ONLY` ([KIP-235](https://cwiki.apache.org/confluence/display/KAFKA/KIP-235%3A+Add+DNS+alias+support+for+secured+connection)) is to handle DNS aliases. How? First, we retrieve all IPs associated with a DNS (`getAllByName`), then perform a reverse DNS lookup (`getCanonicalHostName`) to obtain the corresponding addresses. This ensures that if we have a VIP or DNS alias for multiple brokers, they are all resolved.  

Anyway, the `KafkaProducer` constructor alone reveals a lot about what's happening under the hood. Now, let's take a look at the `send` method.

### send method

```java
    /**
     * Asynchronously send a record to a topic and invoke the provided callback when the send has been acknowledged.
     * <p>
     * The send is asynchronous and this method will return immediately (except for rare cases described below)
     * once the record has been stored in the buffer of records waiting to be sent.
     * This allows sending many records in parallel without blocking to wait for the response after each one.
     * Can block for the following cases: 1) For the first record being sent to 
     * the cluster by this client for the given topic. In this case it will block for up to {@code max.block.ms} milliseconds if 
     * Kafka cluster is unreachable; 2) Allocating a buffer if buffer pool doesn't have any free buffers.
     * <p>
     * The result of the send is a {@link RecordMetadata} specifying the partition the record was sent to, the offset
     * it was assigned and the timestamp of the record. If the producer is configured with acks = 0, the {@link RecordMetadata}
     * will have offset = -1 because the producer does not wait for the acknowledgement from the broker.
     * If {@link org.apache.kafka.common.record.TimestampType#CREATE_TIME CreateTime} is used by the topic, the timestamp
     * will be the user provided timestamp or the record send time if the user did not specify a timestamp for the
     * record. If {@link org.apache.kafka.common.record.TimestampType#LOG_APPEND_TIME LogAppendTime} is used for the
     * topic, the timestamp will be the Kafka broker local time when the message is appended.
     * <p>
     * Since the send call is asynchronous it returns a {@link java.util.concurrent.Future Future} for the
     * {@link RecordMetadata} that will be assigned to this record. Invoking {@link java.util.concurrent.Future#get()
     * get()} on this future will block until the associated request completes and then return the metadata for the record
     * or throw any exception that occurred while sending the record.
     * <p>
     * If you want to simulate a simple blocking call you can call the <code>get()</code> method immediately
     * ...
     **/
    @Override
    public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
        // intercept the record, which can be potentially modified; this method does not throw exceptions
        ProducerRecord<K, V> interceptedRecord = this.interceptors.onSend(record);
        return doSend(interceptedRecord, callback);
    }

```

The method's description is spot on. It tells us that the method is asynchronous but may block if the cluster is unreachable or if there isn't enough memory to allocate a buffer. We also learn that when `acks=0` (AKA "fire and forget"), the producer doesn't expect an acknowledgment from the broker and sets the result offset to `-1` instead of using the actual offset returned by the broker.  

Interceptors act as middleware that take in a record and return either the same record or a modified version. They can do anything from adding headers for telemetry to altering the data.  

After that, `doSend` is invoked. We could just trust it and call it a day—interceptors and `doSend` should be good enough for us.

Jokes aside, here's `doSend` abridged:

```java

        // Append callback takes care of the following:
        //  - call interceptors and user callback on completion
        //  - remember partition that is calculated in RecordAccumulator.append
        AppendCallbacks appendCallbacks = new AppendCallbacks(callback, this.interceptors, record);

        try {
            throwIfProducerClosed();
            // first make sure the metadata for the topic is available
            long nowMs = time.milliseconds();
            ClusterAndWaitTime clusterAndWaitTime;
            try {
                clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), nowMs, maxBlockTimeMs);
            } catch (KafkaException e) {
                if (metadata.isClosed())
                    throw new KafkaException("Producer closed while send in progress", e);
                throw e;
            }
            nowMs += clusterAndWaitTime.waitedOnMetadataMs;
            long remainingWaitMs = Math.max(0, maxBlockTimeMs - clusterAndWaitTime.waitedOnMetadataMs);
            Cluster cluster = clusterAndWaitTime.cluster;
            byte[] serializedKey;
            try {
                serializedKey = keySerializerPlugin.get().serialize(record.topic(), record.headers(), record.key());
            } catch (ClassCastException cce) {
                throw new SerializationException("Can't convert key of class " + record.key().getClass().getName() +
                        " to class " + producerConfig.getClass(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG).getName() +
                        " specified in key.serializer", cce);
            }
            byte[] serializedValue;
            try {
                serializedValue = valueSerializerPlugin.get().serialize(record.topic(), record.headers(), record.value());
            } catch (ClassCastException cce) {
                throw new SerializationException("Can't convert value of class " + record.value().getClass().getName() +
                        " to class " + producerConfig.getClass(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG).getName() +
                        " specified in value.serializer", cce);
            }

            // Try to calculate partition, but note that after this call it can be RecordMetadata.UNKNOWN_PARTITION,
            // which means that the RecordAccumulator would pick a partition using built-in logic (which may
            // take into account broker load, the amount of data produced to each partition, etc.).
            int partition = partition(record, serializedKey, serializedValue, cluster);

            setReadOnly(record.headers());
            Header[] headers = record.headers().toArray();

            int serializedSize = AbstractRecords.estimateSizeInBytesUpperBound(RecordBatch.CURRENT_MAGIC_VALUE,
                    compression.type(), serializedKey, serializedValue, headers);
            ensureValidRecordSize(serializedSize);
            long timestamp = record.timestamp() == null ? nowMs : record.timestamp();

            // Append the record to the accumulator.  Note, that the actual partition may be
            // calculated there and can be accessed via appendCallbacks.topicPartition.
            RecordAccumulator.RecordAppendResult result = accumulator.append(record.topic(), partition, timestamp, serializedKey,
                    serializedValue, headers, appendCallbacks, remainingWaitMs, nowMs, cluster);
            assert appendCallbacks.getPartition() != RecordMetadata.UNKNOWN_PARTITION;

            // Add the partition to the transaction (if in progress) after it has been successfully
            // appended to the accumulator. We cannot do it before because the partition may be
            // unknown. Note that the `Sender` will refuse to dequeue
            // batches from the accumulator until they have been added to the transaction.
            if (transactionManager != null) {
                transactionManager.maybeAddPartition(appendCallbacks.topicPartition());
            }

            if (result.batchIsFull || result.newBatchCreated) {
                log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), appendCallbacks.getPartition());
                this.sender.wakeup();
            }
            return result.future;
            // handling exceptions and record the errors;
            // for API exceptions return them in the future,
            // for other exceptions throw directly
        } catch (ApiException e) {

            // ...

        }
```

We start by creating `AppendCallbacks`, which include both the user-supplied callback and interceptors (whose `onAcknowledgement` method will be invoked). This allows users to interact with the producer request results, whether they succeed or fail.  

For each topic partition we send data to, we need to determine its leader so we can request it to persist our data. That's where `waitOnMetadata` comes in. It issues a Metadata API request to one of the bootstrap servers and caches the response, preventing the need to issue a request for every record.  

Next, the record's key and value are converted from Java objects to bytes using `keySerializerPlugin.get().serialize` and `valueSerializerPlugin.get().serialize`.  

Finally, we determine the record's partition using `partition(record, serializedKey, serializedValue, cluster)`:

```java
    /**
     * computes partition for given record.
     * if the record has partition returns the value otherwise
     * if custom partitioner is specified, call it to compute partition
     * otherwise try to calculate partition based on key.
     * If there is no key or key should be ignored return
     * RecordMetadata.UNKNOWN_PARTITION to indicate any partition
     * can be used (the partition is then calculated by built-in
     * partitioning logic).
     */
    private int partition(ProducerRecord<K, V> record, byte[] serializedKey, byte[] serializedValue, Cluster cluster) {
        if (record.partition() != null)
            return record.partition();

        if (partitionerPlugin.get() != null) {
            int customPartition = partitionerPlugin.get().partition(
                record.topic(), record.key(), serializedKey, record.value(), serializedValue, cluster);
            if (customPartition < 0) {
                throw new IllegalArgumentException(String.format(
                    "The partitioner generated an invalid partition number: %d. Partition number should always be non-negative.", customPartition));
            }
            return customPartition;
        }

        if (serializedKey != null && !partitionerIgnoreKeys) {
            // hash the keyBytes to choose a partition
            return BuiltInPartitioner.partitionForKey(serializedKey, cluster.partitionsForTopic(record.topic()).size());
        } else {
            return RecordMetadata.UNKNOWN_PARTITION;
        }
    }
```

If we have a custom partitioner, we use it. Otherwise, if we have a key and `partitioner.ignore.keys` is false (the default), we rely on the famous key hash by calling `BuiltInPartitioner.partitionForKey`, which under the hood is:

```java
    /*
     * Default hashing function to choose a partition from the serialized key bytes
     */
    public static int partitionForKey(final byte[] serializedKey, final int numPartitions) {
        return Utils.toPositive(Utils.murmur2(serializedKey)) % numPartitions;
    }

```

This is so satisfying! You read about it in various documentation, and it turns out to be exactly as described—getting a partition based on the Murmur2 (a famous hashing algo) key hash.  

However, if there's no key, `UNKNOWN_PARTITION` is returned, and a partition is chosen using a sticky partitioner. This ensures that all partition-less records are grouped into the same partition, allowing for larger batch sizes. The partition selection also considers leader node latency statistics.



```java
    private long batchReady(boolean exhausted, TopicPartition part, Node leader,
                            long waitedTimeMs, boolean backingOff, int backoffAttempts,
                            boolean full, long nextReadyCheckDelayMs, Set<Node> readyNodes) {
        if (!readyNodes.contains(leader) && !isMuted(part)) {
            long timeToWaitMs = backingOff ? retryBackoff.backoff(backoffAttempts > 0 ? backoffAttempts - 1 : 0) : lingerMs;
            boolean expired = waitedTimeMs >= timeToWaitMs;
            boolean transactionCompleting = transactionManager != null && transactionManager.isCompleting();
            boolean sendable = full
                    || expired
                    || exhausted
                    || closed
                    || flushInProgress()
                    || transactionCompleting;
            if (sendable && !backingOff) {
                readyNodes.add(leader);
            } else {
                long timeLeftMs = Math.max(timeToWaitMs - waitedTimeMs, 0);
                // Note that this results in a conservative estimate since an un-sendable partition may have
                // a leader that will later be found to have sendable data. However, this is good enough
                // since we'll just wake up and then sleep again for the remaining time.
                nextReadyCheckDelayMs = Math.min(timeLeftMs, nextReadyCheckDelayMs);
            }
        }
        return nextReadyCheckDelayMs;
    }
```


After that we pass the ball to the `RecordAccumulator` using `accumulator.append` and it will takes care of allocating a buffer for each batch and adding the record to it.


## RecordAccumulator
The class documentation reads:

```java
/**
 * This class acts as a queue that accumulates records into {@link MemoryRecords}
 * instances to be sent to the server.
 * <p>
 * The accumulator uses a bounded amount of memory and append calls will block when that memory is exhausted, unless
 * this behavior is explicitly disabled.
 */
```
and the object is instantiated within the `KafkaProducer`'s constructor:

```java 
this.accumulator = new RecordAccumulator(logContext,
                    batchSize,
                    compression,
                    lingerMs(config),
                    retryBackoffMs,
                    retryBackoffMaxMs,
                    deliveryTimeoutMs,
                    partitionerConfig,
                    metrics,
                    PRODUCER_METRIC_GROUP_NAME,
                    time,
                    apiVersions,
                    transactionManager,
                    new BufferPool(this.totalMemorySize, batchSize, metrics, time, PRODUCER_METRIC_GROUP_NAME));
```

This is where batching takes place. Where the tradeoff between `batch.size` and `linger.ms` is implemented. Where retries are made. And where a produce attempt is timed out after `deliveryTimeoutMs` (defaults to 2 min).

The producer's `doSend` calls the Accumulator's `append` method:

```java
    public RecordAppendResult append(String topic,
                                     int partition,
                                     long timestamp,
                                     byte[] key,
                                     byte[] value,
                                     Header[] headers,
                                     AppendCallbacks callbacks,
                                     long maxTimeToBlock,
                                     long nowMs,
                                     Cluster cluster) throws InterruptedException {
        TopicInfo topicInfo = topicInfoMap.computeIfAbsent(topic, k -> new TopicInfo(createBuiltInPartitioner(logContext, k, batchSize)));

        // We keep track of the number of appending thread to make sure we do not miss batches in
        // abortIncompleteBatches().
        appendsInProgress.incrementAndGet();
        ByteBuffer buffer = null;
        if (headers == null) headers = Record.EMPTY_HEADERS;
        try {
            // Loop to retry in case we encounter partitioner's race conditions.
            while (true) {
                // If the message doesn't have any partition affinity, so we pick a partition based on the broker
                // availability and performance.  Note, that here we peek current partition before we hold the
                // deque lock, so we'll need to make sure that it's not changed while we were waiting for the
                // deque lock.
                final BuiltInPartitioner.StickyPartitionInfo partitionInfo;
                final int effectivePartition;
                if (partition == RecordMetadata.UNKNOWN_PARTITION) {
                    partitionInfo = topicInfo.builtInPartitioner.peekCurrentPartitionInfo(cluster);
                    effectivePartition = partitionInfo.partition();
                } else {
                    partitionInfo = null;
                    effectivePartition = partition;
                }

                // Now that we know the effective partition, let the caller know.
                setPartition(callbacks, effectivePartition);

                // check if we have an in-progress batch
                Deque<ProducerBatch> dq = topicInfo.batches.computeIfAbsent(effectivePartition, k -> new ArrayDeque<>());
                synchronized (dq) {
                    // After taking the lock, validate that the partition hasn't changed and retry.
                    if (partitionChanged(topic, topicInfo, partitionInfo, dq, nowMs, cluster))
                        continue;

                    RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callbacks, dq, nowMs);
                    if (appendResult != null) {
                        // If queue has incomplete batches we disable switch (see comments in updatePartitionInfo).
                        boolean enableSwitch = allBatchesFull(dq);
                        topicInfo.builtInPartitioner.updatePartitionInfo(partitionInfo, appendResult.appendedBytes, cluster, enableSwitch);
                        return appendResult;
                    }
                }

                if (buffer == null) {
                    int size = Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(
                            RecordBatch.CURRENT_MAGIC_VALUE, compression.type(), key, value, headers));
                    log.trace("Allocating a new {} byte message buffer for topic {} partition {} with remaining timeout {}ms", size, topic, effectivePartition, maxTimeToBlock);
                    // This call may block if we exhausted buffer space.
                    buffer = free.allocate(size, maxTimeToBlock);
                    // Update the current time in case the buffer allocation blocked above.
                    // NOTE: getting time may be expensive, so calling it under a lock
                    // should be avoided.
                    nowMs = time.milliseconds();
                }

                synchronized (dq) {
                    // After taking the lock, validate that the partition hasn't changed and retry.
                    if (partitionChanged(topic, topicInfo, partitionInfo, dq, nowMs, cluster))
                        continue;

                    RecordAppendResult appendResult = appendNewBatch(topic, effectivePartition, dq, timestamp, key, value, headers, callbacks, buffer, nowMs);
                    // Set buffer to null, so that deallocate doesn't return it back to free pool, since it's used in the batch.
                    if (appendResult.newBatchCreated)
                        buffer = null;
                    // If queue has incomplete batches we disable switch (see comments in updatePartitionInfo).
                    boolean enableSwitch = allBatchesFull(dq);
                    topicInfo.builtInPartitioner.updatePartitionInfo(partitionInfo, appendResult.appendedBytes, cluster, enableSwitch);
                    return appendResult;
                }
            }
        } finally {
            free.deallocate(buffer);
            appendsInProgress.decrementAndGet();
        }
    }

```
We start with `TopicInfo topicInfo = topicInfoMap.computeIfAbsent(topic, k -> new TopicInfo(createBuiltInPartitioner(logContext, k, batchSize)));`, in my opinion, `topicInfoMap` is the most important variable in this whole class. Here is its init code followed by the `TopicInfo` class:

```java
    private final ConcurrentMap<String /*topic*/, TopicInfo> topicInfoMap = new CopyOnWriteMap<>();

    /**
     * Per topic info.
     */
    private static class TopicInfo {
        public final ConcurrentMap<Integer /*partition*/, Deque<ProducerBatch>> batches = new CopyOnWriteMap<>();
        public final BuiltInPartitioner builtInPartitioner;

        public TopicInfo(BuiltInPartitioner builtInPartitioner) {
            this.builtInPartitioner = builtInPartitioner;
        }
    }
```
We maintain a `ConcurrentMap` keyed by topic, where each value is a `TopicInfo` object. This object, in turn, holds another `ConcurrentMap` keyed by partition, with values being a `Deque` (double-ended queue) of batches. The core responsibility of `RecordAccumulator` is to allocate memory for these record batches and fill them with records, either until `linger.ms` is reached or the batch reaches its `batch.size` limit.  

Notice how we use `computeIfAbsent` to retrieve the `TopicInfo`, and later use it again to get the `ProducerBatch` deque:  

```java
// Check if we have an in-progress batch  
Deque<ProducerBatch> dq = topicInfo.batches.computeIfAbsent(effectivePartition, k -> new ArrayDeque<>());
```  

This `computeIfAbsent` call is at the heart of the Kafka Producer batching mechanism. The `send` method ultimately calls `append`, and within it, there's a map that holds another map, which holds a queue of batches for each partition. As long as a batch remains open (i.e. not older than `linger.ms` and not full up to `batch.size`), it's reused and new records are appended to it and batched together.  

Once we retrieve `topicInfo` and increment the `appendsInProgress` counter-used to abort batches in case of errors—we enter an infinite loop. This loop either exits with a return or an exception. It's necessary because the target partition might change while we're inside the loop. Remember, the Kafka Producer is designed for a multi-threaded environment and is considered thread-safe. Additionally, the batch we're trying to append to might become full or not have enough space, requiring a retry.  

Inside the loop, if the record has an `UNKNOWN_PARTITION` (meaning there's no custom partitioner and no key-based partitioning), a sticky partition is selected using `builtInPartitioner.peekCurrentPartitionInfo`, based on broker availability and performance stats.  

At this point, we have the partition's `Deque<ProducerBatch>`, and we use `synchronized (dq)` to ensure no other threads interfere. Then, `tryAppend` is called:

```java
    /**
     *  Try to append to a ProducerBatch.
     *
     *  If it is full, we return null and a new batch is created. We also close the batch for record appends to free up
     *  resources like compression buffers. The batch will be fully closed (ie. the record batch headers will be written
     *  and memory records built) in one of the following cases (whichever comes first): right before send,
     *  if it is expired, or when the producer is closed.
     */
    private RecordAppendResult tryAppend(long timestamp, byte[] key, byte[] value, Header[] headers,
                                         Callback callback, Deque<ProducerBatch> deque, long nowMs) {
        if (closed)
            throw new KafkaException("Producer closed while send in progress");
        ProducerBatch last = deque.peekLast();
        if (last != null) {
            int initialBytes = last.estimatedSizeInBytes();
            FutureRecordMetadata future = last.tryAppend(timestamp, key, value, headers, callback, nowMs);
            if (future == null) {
                last.closeForRecordAppends();
            } else {
                int appendedBytes = last.estimatedSizeInBytes() - initialBytes;
                return new RecordAppendResult(future, deque.size() > 1 || last.isFull(), false, appendedBytes);
            }
        }
        return null;
    }
```
If the producer is not closed and there's a producer batch in the queue, we attempt to append to it. If appending fails (`future == null`), we close the batch so it can be sent and removed from the queue. If it succeeds, we return a `RecordAppendResult` object.  

Now, let's look at `if (buffer == null)` inside `append`. This condition is met if the dequeue had no `RecordBatch` or if appending to an existing `RecordBatch` failed. In that case, we allocate a new buffer using `free.allocate`. This allocation process is quite interesting, and we'll dive into it in the upcoming `BufferPool` section.  

After allocating the buffer, `appendNewBatch` is called to create a new batch and add it to the queue. But before doing so, it first checks whether another thread has already created a new batch:

```java
      // Inside  private RecordAppendResult appendNewBatch
      RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callbacks, dq, nowMs);
        if (appendResult != null) {
            // Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...
            return appendResult;
        }

        MemoryRecordsBuilder recordsBuilder = recordsBuilder(buffer);
        ProducerBatch batch = new ProducerBatch(new TopicPartition(topic, partition), recordsBuilder, nowMs);
        FutureRecordMetadata future = Objects.requireNonNull(batch.tryAppend(timestamp, key, value, headers,
                callbacks, nowMs));

        dq.addLast(batch);
```

The `// Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...` comment is just a sight for sore eyes. When it comes to multithreading, hope is all we got.

After the batch append, we call `builtInPartitioner.updatePartitionInfo` which might change the sticky partition.

Finally, if the allocated buffer has not been successfully used in a new batch, it will be deallocated to free up memory.


## The Magical BufferPool
So, `KafkaProducer` calls its `send` method, which in turn calls `RecordAccumulator`'s `append` method. This method is responsible for adding records to an in-memory batch until they are sent.  

But how much memory do we use? How do we free memory? How do we manage each batch's memory to ensure that no one is starved while also staying within the producer's configured memory limit (`buffer.memory`, which defaults to 32 MB)?  

The answers to all these questions lead us to the `BufferPool` class. Its documentation reads:

```java
/**
 * A pool of ByteBuffers kept under a given memory limit. This class is fairly specific to the needs of the producer. In
 * particular it has the following properties:
 * <ol>
 * <li>There is a special "poolable size" and buffers of this size are kept in a free list and recycled
 * <li>It is fair. That is all memory is given to the longest waiting thread until it has sufficient memory. This
 * prevents starvation or deadlock when a thread asks for a large chunk of memory and needs to block until multiple
 * buffers are deallocated.
 * </ol>
 */
```
The `BufferPool`'s constructor has two key args:

```java
/**
 * Create a new buffer pool
 *
 * @param memory The maximum amount of memory that this buffer pool can allocate
 * @param poolableSize The buffer size to cache in the free list rather than deallocating
 *...
 */
public BufferPool(long memory, int poolableSize, ...)
```

The `BufferPool` is [initialized](https://github.com/apache/kafka/blob/5d2bfb4151700000fc5ec22918ecd3cecb5178a5/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java#L440) all the way in the `KafkaProducer`:

```java
new BufferPool(this.totalMemorySize, batchSize, ...)
```

`totalMemorySize` is the famous `buffer.memory` which is the total amount of memory the producer can use to buffer records and `batchSize` is arguably the most famous producer config i.e. `batch.size`.

The `RecordAccumulator` tries to [allocate buffer](https://github.com/apache/kafka/blob/5d2bfb4151700000fc5ec22918ecd3cecb5178a5/clients/src/main/java/org/apache/kafka/clients/producer/internals/RecordAccumulator.java#L341):

```java
int size = Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(
        RecordBatch.CURRENT_MAGIC_VALUE, compression.type(), key, value, headers));
log.trace("Allocating a new {} byte message buffer for topic {} partition {} with remaining timeout {}ms", size, topic, effectivePartition, maxTimeToBlock);
// This call may block if we exhausted buffer space.
buffer = free.allocate(size, maxTimeToBlock);
```

Notice the `max`: the buffer'size can exceed `batch.size` if we have a record that exceeds that size. 

`free` is our `BufferPool` and we try try to allocate our ByteBuffer and wait up to `maxTimeToBlock` which is none other than `max.block.ms` (minus any time it took to get the topic's metadata if it was not already present).

So, how does `allocate` work? 

We start by making sanity checks and acquiring the lock and making sure there is no contention with other thread:

```java
        if (size > this.totalMemory)
            throw new IllegalArgumentException("Attempt to allocate " + size
                                               + " bytes, but there is a hard limit of "
                                               + this.totalMemory
                                               + " on memory allocations.");

        ByteBuffer buffer = null;
        this.lock.lock();

        if (this.closed) {
            this.lock.unlock();
            throw new KafkaException("Producer closed while allocating memory");
        }
```

if `BufferPool` allocates a buffer of size `batch.size`, it will add it a `Deque<ByteBuffer>` aptly called `free` when it gets freed. If the buffer's size is different than `batch.size`, it's unlikely another request will need a similar one and it is thus simply discarded and we account for that in `nonPooledAvailableMemory`. Most allocation requests will be for the configured `batch.size`, unless there is a message that exceeds that. 

```java
// check if we have a free buffer of the right size pooled
if (size == poolableSize && !this.free.isEmpty())
    return this.free.pollFirst();

```
After acquiring the lock, if the request size is equal to `batch.size` and there is an available buffer in our `free` deque, we simply grab it and return.

If not, we check if there is enough memory (pooled and unpooled), and if necessary, `freeUp` will free some buffer from the `free` deque:
```java
 // now check if the request is immediately satisfiable with the
// memory on hand or if we need to block
int freeListSize = freeSize() * this.poolableSize;
 if (this.nonPooledAvailableMemory + freeListSize >= size) {
                // we have enough unallocated or pooled memory to immediately
                // satisfy the request, but need to allocate the buffer
                freeUp(size);
                this.nonPooledAvailableMemory -= size;
            }
```

`freeUp` is simply grabbing buffers from `free` until we have enough memory:

```java
    /**
     * Attempt to ensure we have at least the requested number of bytes of memory for allocation by deallocating pooled
     * buffers (if needed)
     */
    private void freeUp(int size) {
        while (!this.free.isEmpty() && this.nonPooledAvailableMemory < size)
            this.nonPooledAvailableMemory += this.free.pollLast().capacity();
    }
```
If there is enough memory for `size`, great! We simply call `ByteBuffer.allocate` (Java's native allocation) using `safeAllocateByteBuffer` which adds a nice try to protect against OOM.

However, if we were unable to find enough memory, we enter into the following loop:

```java
          Condition moreMemory = this.lock.newCondition();
          while (accumulated < size) {
                        long startWaitNs = time.nanoseconds();
                        long timeNs;
                        boolean waitingTimeElapsed;
                        try {
                            waitingTimeElapsed = !moreMemory.await(remainingTimeToBlockNs, TimeUnit.NANOSECONDS);
                        } finally {
                            long endWaitNs = time.nanoseconds();
                            timeNs = Math.max(0L, endWaitNs - startWaitNs);
                            recordWaitTime(timeNs);
                        }

                       if (waitingTimeElapsed) {
                            this.metrics.sensor("buffer-exhausted-records").record();
                            throw new BufferExhaustedException("Failed to allocate " + size + " bytes within the configured max blocking time "
                                + maxTimeToBlockMs + " ms. Total memory: " + totalMemory() + " bytes. Available memory: " + availableMemory()
                                + " bytes. Poolable size: " + poolableSize() + " bytes");
                        }

                        remainingTimeToBlockNs -= timeNs;
                        // check if we can satisfy this request from the free list,
                        // otherwise allocate memory
                        if (accumulated == 0 && size == this.poolableSize && !this.free.isEmpty()) {
                            // just grab a buffer from the free list
                            buffer = this.free.pollFirst();
                            accumulated = size;
                        } else {
                            // we'll need to allocate memory, but we may only get
                            // part of what we need on this iteration
                            freeUp(size - accumulated);
                            int got = (int) Math.min(size - accumulated, this.nonPooledAvailableMemory);
                            this.nonPooledAvailableMemory -= got;
                            accumulated += got;
                        }
                    }
```
The loop uses `Condition` to put the thread to sleep and release the lock. The idea is to wait for another thread to free up memory and call `signal()`, waking up our sleeping thread.  

If our thread stops waiting because it has waited too long, an exception is thrown. Otherwise, it means another thread has freed some memory, so we check if a buffer is available in `free` or call `freeUp` to reclaim memory.  

This `while` loop exits when enough memory has been accumulated or after `maxTimeToBlockMs` has elapsed.  

After the loop, we encounter this nifty `finally` block:

```java
this.waiters.remove(moreMemory);
//...
finally {
            // signal any additional waiters if there is more memory left
            // over for them
            try {
                if (!(this.nonPooledAvailableMemory == 0 && this.free.isEmpty()) && !this.waiters.isEmpty())
                    this.waiters.peekFirst().signal();
            } finally {
                // Another finally... otherwise find bugs complains
                lock.unlock();
            }
        }
```
`this.waiters` is a `Deque<Condition>`, where all `allocate` calls that are waiting for memory get added.  

Before exiting, we first remove the current thread's `Condition` from `waiters`, since it's no longer waiting for memory. Then, if there's any free memory available, we call `signal()` on the first waiter (following FIFO semantics) to wake it up and allow it to proceed.  

There's also a `deallocate` method, which is called by `RecordAccumulator` once it's done with the buffer.

```java
    /**
     * Return buffers to the pool. If they are of the poolable size add them to the free list, otherwise just mark the
     * memory as free.
     *
     * @param buffer The buffer to return
     * @param size The size of the buffer to mark as deallocated, note that this may be smaller than buffer.capacity
     *             since the buffer may re-allocate itself during in-place compression
     */
    public void deallocate(ByteBuffer buffer, int size) {
        lock.lock();
        try {
            if (size == this.poolableSize && size == buffer.capacity()) {
                buffer.clear();
                this.free.add(buffer);
            } else {
                this.nonPooledAvailableMemory += size;
            }
            Condition moreMem = this.waiters.peekFirst();
            if (moreMem != null)
                moreMem.signal();
        } finally {
            lock.unlock();
        }
    }
```
We acquire the lock, and if the buffer is of size `batch.size`, we clear it and add it to the `free` deque. If it's not, we simply let the buffer be garbage collected and account for it in `nonPooledAvailableMemory`.  

Most importantly, we call `signal()` on the first waiter (if there's one), waking it up so it can use the memory we just freed.  

And there you have it! That's how the producer manages different buffers for the record batches to be sent.  

These lines are executed for every Kafka-produced batch, meaning they run billions of times every second. The impact is truly tremendous!

## Return to Sender 

We return to the `Sender` thread. Before that, I'd like to note that ["Return to Sender"](https://youtu.be/LZmUfUBqE-s?si=dLuSBHC2MLXawZJm) is also the name of a tremendous Elvis Presley song. You might want to give it a listen. 

`Sender` is simply a Runnable:

```java
/**
 * The background thread that handles the sending of produce requests to the Kafka cluster. This thread makes metadata
 * requests to renew its view of the cluster and then sends produce requests to the appropriate nodes.
 */
public class Sender implements Runnable 
```

its main loop:

```java
     // main loop, runs until close is called
        while (running) {
            try {
                runOnce();
            } catch (Exception e) {
                log.error("Uncaught error in kafka producer I/O thread: ", e);
            }
        }
```

How about this `runOnce`? Here it is minus the transactional stuff:

```java

        long currentTimeMs = time.milliseconds();
        long pollTimeout = sendProducerData(currentTimeMs);
        client.poll(pollTimeout, currentTimeMs); //
```
`client` is the network client and `poll` is to read and write data from the network sockets.

The meat and potatoes are in `sendProducerData`:

```java
    private long sendProducerData(long now) {
        MetadataSnapshot metadataSnapshot = metadata.fetchMetadataSnapshot();
        // get the list of partitions with data ready to send
        RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(metadataSnapshot, now);

        // ...

        // create produce requests
        Map<Integer, List<ProducerBatch>> batches = this.accumulator.drain(metadataSnapshot, result.readyNodes, this.maxRequestSize, now);

        //...

        sendProduceRequests(batches, now);
    } 
```
We start by calling the `ready` method of the accumulator and supplying it with a `metadataSnapshot` ( basically a list of the brokers, available partitions and leaders, etc).

```java

    /**
     * Get a list of nodes whose partitions are ready to be sent, and the earliest time at which any non-sendable
     * partition will be ready; Also return the flag for whether there are any unknown leaders for the accumulated
     * partition batches.
     * <p>
     * A destination node is ready to send data if:
     * <ol>
     * <li>There is at least one partition that is not backing off its send
     * <li><b>and</b> those partitions are not muted (to prevent reordering if
     *   {@value org.apache.kafka.clients.producer.ProducerConfig#MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION}
     *   is set to one)</li>
     * <li><b>and <i>any</i></b> of the following are true</li>
     * <ul>
     *     <li>The record set is full</li>
     *     <li>The record set has sat in the accumulator for at least lingerMs milliseconds</li>
     *     <li>The accumulator is out of memory and threads are blocking waiting for data (in this case all partitions
     *     are immediately considered ready).</li>
     *     <li>The accumulator has been closed</li>
     * </ul>
     * </ol>
     */
    public ReadyCheckResult ready(MetadataSnapshot metadataSnapshot, long nowMs) {
        Set<Node> readyNodes = new HashSet<>();
        long nextReadyCheckDelayMs = Long.MAX_VALUE;
        Set<String> unknownLeaderTopics = new HashSet<>();
        // Go topic by topic so that we can get queue sizes for partitions in a topic and calculate
        // cumulative frequency table (used in partitioner).
        for (Map.Entry<String, TopicInfo> topicInfoEntry : this.topicInfoMap.entrySet()) {
            final String topic = topicInfoEntry.getKey();
            nextReadyCheckDelayMs = partitionReady(metadataSnapshot, nowMs, topic, topicInfoEntry.getValue(), nextReadyCheckDelayMs, readyNodes, unknownLeaderTopics);
        }
        return new ReadyCheckResult(readyNodes, nextReadyCheckDelayMs, unknownLeaderTopics);
    }

```

This is it! This is where partitions and batches are deemed ready for sending. Where `batch.size` and `linger.ms` can shine.
We loop over the `topicInfoMap` entries, and for each topic we call `partitionReady`:

```java
// inside private long partitionReady

 boolean exhausted = this.free.queued() > 0; // threads waiting for memory
        for (Map.Entry<Integer, Deque<ProducerBatch>> entry : batches.entrySet()) {
            Deque<ProducerBatch> deque = entry.getValue();

                // ...

            synchronized (deque) {
                // Deques are often empty in this path, esp with large partition counts,
                // so we exit early if we can.
                ProducerBatch batch = deque.peekFirst();
                if (batch == null) {
                    continue;
                }

                waitedTimeMs = batch.waitedTimeMs(nowMs);
                batch.maybeUpdateLeaderEpoch(leaderEpoch);
                backingOff = shouldBackoff(batch.hasLeaderChangedForTheOngoingRetry(), batch, waitedTimeMs);
                backoffAttempts = batch.attempts();
                dequeSize = deque.size();
                full = dequeSize > 1 || batch.isFull();
            }

             if (leader == null) {
                // This is a partition for which leader is not known, but messages are available to send.
                // Note that entries are currently not removed from batches when deque is empty.
                unknownLeaderTopics.add(part.topic());
            }
            // ...
             nextReadyCheckDelayMs = batchReady(exhausted, part, leader, waitedTimeMs, backingOff,
                    backoffAttempts, full, nextReadyCheckDelayMs, readyNodes);
        }

```
We get information about the batch. Notably, `waitedTimeMs` which is how long has batch waited (`now - createTime`) and whether the batch is full. Then we call `batchReady`:

```java
    private long batchReady(boolean exhausted, TopicPartition part, Node leader,
                            long waitedTimeMs, boolean backingOff, int backoffAttempts,
                            boolean full, long nextReadyCheckDelayMs, Set<Node> readyNodes) {
        if (!readyNodes.contains(leader) && !isMuted(part)) {
            long timeToWaitMs = backingOff ? retryBackoff.backoff(backoffAttempts > 0 ? backoffAttempts - 1 : 0) : lingerMs;
            boolean expired = waitedTimeMs >= timeToWaitMs;
            boolean transactionCompleting = transactionManager != null && transactionManager.isCompleting();
            boolean sendable = full
                    || expired
                    || exhausted
                    || closed
                    || flushInProgress()
                    || transactionCompleting;
            if (sendable && !backingOff) {
                readyNodes.add(leader);
            } else {
                long timeLeftMs = Math.max(timeToWaitMs - waitedTimeMs, 0);
                // Note that this results in a conservative estimate since an un-sendable partition may have
                // a leader that will later be found to have sendable data. However, this is good enough
                // since we'll just wake up and then sleep again for the remaining time.
                nextReadyCheckDelayMs = Math.min(timeLeftMs, nextReadyCheckDelayMs);
            }
        }
        return nextReadyCheckDelayMs;
    }
```
Notice `timeToWaitMs`. If we're not backing off due to a retry, we wait for `lingerMs`. A batch is considered `expired` if the time we've already waited (`now - createTime`) exceeds `lingerMs`.  

A batch is sendable if:  
- It has expired (we waited longer than `linger.ms`).  
- It is full (`batch.size` bytes reached).  
- Buffer memory is exhausted (other threads are waiting).  
- `flush()` has been called in `KafkaProducer`.  
- A transaction is completing.  

After `ready` runs, `readyNodes` is populated with all nodes that have batches ready to be sent. Then, we call `drain`, which collects the batches from those nodes. For each topic partition, batches are bundled into a request, which is then handed off to the network client. Upon receiving a response, the appropriate callbacks are triggered, and the cycle continues.  

One final observation: the `Sender` thread and the various application threads running `RecordAccumulator` are synchronized using `synchronized (dq)`. A simple way to think about this is that the `KafkaProducer` adds records to `RecordBatches` queues via the `RecordAccumulator`, while the `Sender` thread continuously checks for sendable batches (based on various properties:`linger.ms`, `batch.size`, etc) and passes them to the network client.

## Conclusion
This is nothing but a small snapshot into the innards of the Kafka Producer. A flurry of amazing technical details and smart design decisions have been passed by in this post. I'm also just sharing my understanding while going through the code. If I have misunderstood or got something wrong, please let me know.

If you want to get in touch, you can find me on Twitter at [@moncef_abboud](https://x.com/moncef_abboud).  
