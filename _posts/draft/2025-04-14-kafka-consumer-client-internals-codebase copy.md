---
title: "A Look Under the Hood of Kafka Consumer"
author: cef
date: 2025-04-14
categories: [Technical Writing, Open Source]
tags: [Apache Kafka, Open Source]
render_with_liquid: false
description: Exploring the code behind the Kafka Consumer client.
---

## Intro
In my last post I explored the Kafka Producer internals and it was a very fun and informative exercise. That effort naturally led me to take look into the Kafka Consumer codebase, only the classic one, which manages consumer groups on the consumer side. There is a new `group.protocol` introduced in KIP-848 which moves the burden of group management from a the leader consumer (the first one to join the consumer group) to the broker that coordinates that group AKA the coordinator.

## KafkaConsumer Class
As we did in the previous post, let's look at a basic example for a plain Java Kafka Consumer:

```java

        Properties properties = new Properties();
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "broker:9092");
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, "test-group");
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
        // Create Kafka consumer
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(properties);
        // Subscribe to the topic(s)
        consumer.subscribe(Collections.singletonList("test-topic"));
        // Poll and consume messages
        try {
            while (true) {
                var records = consumer.poll(1000);  // Poll for new messages with timeout of 1 second
                // Process the messages
                records.forEach(record -> {
                    System.out.printf("Consumed record with key: %s, value: %s%n", record.key(), record.value());
                });
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            // Always close the consumer
            consumer.close();
        }

```

Nothing too complicated. We need a broker to read data from. Specify a consumer group and a topic we'd like to consume. Call the magical `poll` method (the workhorse of this operation) and loop over those records and do some processing. In our case, we're doing a fancy `printf`, the peek of sophistication. 😉


So the public API to the consumer is [`KafkaConsumer`](https://github.com/apache/kafka/blob/8c77953d5fa84ce1dfdf83f73560444a4acabc1f/clients/src/main/java/org/apache/kafka/clients/consumer/KafkaConsumer.java). 

If we strip the Javadoc comments, it's rather a short class. It's actually just a facade to different implementations of the consumer either the classical one or the new kip-848 one. 

Let's follow our constructor path:

```java

// we call this first
    public KafkaConsumer(Properties properties) {
        this(properties, null, null);
    }

// which then calls: 

    public KafkaConsumer(Properties properties,
                         Deserializer<K> keyDeserializer,
                         Deserializer<V> valueDeserializer) {
        this(propsToMap(properties), keyDeserializer, valueDeserializer);
    }

// which calls:

    public KafkaConsumer(Map<String, Object> configs,
                         Deserializer<K> keyDeserializer,
                         Deserializer<V> valueDeserializer) {
        this(new ConsumerConfig(ConsumerConfig.appendDeserializerToConfig(configs, keyDeserializer, valueDeserializer)),
                keyDeserializer, valueDeserializer);
    }

// and finally:

    KafkaConsumer(ConsumerConfig config, Deserializer<K> keyDeserializer, Deserializer<V> valueDeserializer) {
        delegate = CREATOR.create(config, keyDeserializer, valueDeserializer);
    }
```

Aha! We pass the ball to this `delegate`, which will  be called in all other important methods. For instance, the famous `poll` method in `KafkaConsumer` is nothing but:

```java
    public ConsumerRecords<K, V> poll(final Duration timeout) {
        return delegate.poll(timeout);
    }
```
So this `delegate` is what we are actually interested in. `delegate` is of type `ConsumerDelegate<K, V>` which is an interface that has all the `KafkaConsumer` methods. 

`CREATOR` is an instance of `ConsumerDelegateCreator` which is basically a factor and its `create` method used to instantiate the `delegate` is short and sweet:

```java

    public <K, V> ConsumerDelegate<K, V> create(ConsumerConfig config,
                                                Deserializer<K> keyDeserializer,
                                                Deserializer<V> valueDeserializer) {
        try {
            GroupProtocol groupProtocol = GroupProtocol.valueOf(config.getString(ConsumerConfig.GROUP_PROTOCOL_CONFIG).toUpperCase(Locale.ROOT));

            if (groupProtocol == GroupProtocol.CONSUMER)
                return new AsyncKafkaConsumer<>(config, keyDeserializer, valueDeserializer);
            else
                return new ClassicKafkaConsumer<>(config, keyDeserializer, valueDeserializer);
        } catch (KafkaException e) {
            throw e;
        } catch (Throwable t) {
            throw new KafkaException("Failed to construct Kafka consumer", t);
        }
    }
```

The comment above it reads:

```
 * The current logic for the {@code ConsumerCreator} inspects the incoming configuration and determines if
 * it is using the new consumer group protocol (KIP-848) or if it should fall back to the existing, legacy group
 * protocol. This is based on the presence and value of the {@link ConsumerConfig#GROUP_PROTOCOL_CONFIG group.protocol}
 * configuration. If the value is present and equal to &quot;{@code consumer}&quot;, the {@link AsyncKafkaConsumer}
 * will be returned. Otherwise, the {@link ClassicKafkaConsumer} will be returned.
```

so if we have `group.protocol=consumer`, we use the new `AsyncKafkaConsumer` KIP-848 consumer, otherwise we default to `ClassicKafkaConsumer`. 

In this post, let's stick to the classic consumer. 

## ClassicKafkaConsumer 
I like to listen to classical music when working sometimes, it's can be very calming. Orchestras come to mind when I say  `ClassicKafkaConsumer`. I also think about the movie Whiplash, which is fantastic!

Anyway, let's take a look at the constructor (abridged):

```java
    ClassicKafkaConsumer(ConsumerConfig config, Deserializer<K> keyDeserializer, Deserializer<V> valueDeserializer) {
        try {
            GroupRebalanceConfig groupRebalanceConfig = new GroupRebalanceConfig(config,
                    GroupRebalanceConfig.ProtocolType.CONSUMER);

            this.groupId = Optional.ofNullable(groupRebalanceConfig.groupId);
            this.clientId = config.getString(CommonClientConfigs.CLIENT_ID_CONFIG);
            LogContext logContext = createLogContext(config, groupRebalanceConfig);
            this.log = logContext.logger(getClass());
            boolean enableAutoCommit = config.getBoolean(ENABLE_AUTO_COMMIT_CONFIG);

            log.debug("Initializing the Kafka consumer");
            this.requestTimeoutMs = config.getInt(ConsumerConfig.REQUEST_TIMEOUT_MS_CONFIG);
            this.defaultApiTimeoutMs = config.getInt(ConsumerConfig.DEFAULT_API_TIMEOUT_MS_CONFIG);
            this.time = Time.SYSTEM;
            List<MetricsReporter> reporters = CommonClientConfigs.metricsReporters(clientId, config);
            this.clientTelemetryReporter = CommonClientConfigs.telemetryReporter(clientId, config);
            this.clientTelemetryReporter.ifPresent(reporters::add);
            this.metrics = createMetrics(config, time, reporters);
            this.retryBackoffMs = config.getLong(ConsumerConfig.RETRY_BACKOFF_MS_CONFIG);
            this.retryBackoffMaxMs = config.getLong(ConsumerConfig.RETRY_BACKOFF_MAX_MS_CONFIG);

            List<ConsumerInterceptor<K, V>> interceptorList = configuredConsumerInterceptors(config);
            this.interceptors = new ConsumerInterceptors<>(interceptorList, metrics);
            this.deserializers = new Deserializers<>(config, keyDeserializer, valueDeserializer, metrics);
            this.subscriptions = createSubscriptionState(config, logContext);
            ClusterResourceListeners clusterResourceListeners = ClientUtils.configureClusterResourceListeners(
                    metrics.reporters(),
                    interceptorList,
                    Arrays.asList(this.deserializers.keyDeserializer(), this.deserializers.valueDeserializer()));
            this.metadata = new ConsumerMetadata(config, subscriptions, logContext, clusterResourceListeners);
            List<InetSocketAddress> addresses = ClientUtils.parseAndValidateAddresses(config);
            this.metadata.bootstrap(addresses);

            FetchMetricsManager fetchMetricsManager = createFetchMetricsManager(metrics);
            FetchConfig fetchConfig = new FetchConfig(config);
            this.isolationLevel = fetchConfig.isolationLevel;

            ApiVersions apiVersions = new ApiVersions();
            this.client = createConsumerNetworkClient(config,
                    metrics,
                    logContext,
                    apiVersions,
                    time,
                    metadata,
                    fetchMetricsManager.throttleTimeSensor(),
                    retryBackoffMs,
                    clientTelemetryReporter.map(ClientTelemetryReporter::telemetrySender).orElse(null));

            this.assignors = ConsumerPartitionAssignor.getAssignorInstances(
                    config.getList(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG),
                    config.originals(Collections.singletonMap(ConsumerConfig.CLIENT_ID_CONFIG, clientId))
            );

            // no coordinator will be constructed for the default (null) group id
            if (groupId.isEmpty()) {
                config.ignore(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG);
                config.ignore(THROW_ON_FETCH_STABLE_OFFSET_UNSUPPORTED);
                this.coordinator = null;
            } else {
                this.coordinator = new ConsumerCoordinator(groupRebalanceConfig,
                        logContext,
                        this.client,
                        assignors,
                        this.metadata,
                        this.subscriptions,
                        metrics,
                        CONSUMER_METRIC_GROUP_PREFIX,
                        this.time,
                        enableAutoCommit,
                        config.getInt(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG),
                        this.interceptors,
                        config.getBoolean(THROW_ON_FETCH_STABLE_OFFSET_UNSUPPORTED),
                        config.getString(ConsumerConfig.CLIENT_RACK_CONFIG),
                        clientTelemetryReporter);
            }
            this.fetcher = new Fetcher<>(
                    logContext,
                    this.client,
                    this.metadata,
                    this.subscriptions,
                    fetchConfig,
                    this.deserializers,
                    fetchMetricsManager,
                    this.time,
                    apiVersions);
            this.offsetFetcher = new OffsetFetcher(logContext,
                    client,
                    metadata,
                    subscriptions,
                    time,
                    retryBackoffMs,
                    requestTimeoutMs,
                    isolationLevel,
                    apiVersions);
            this.topicMetadataFetcher = new TopicMetadataFetcher(logContext,
                    client,
                    retryBackoffMs,
                    retryBackoffMaxMs);

            log.debug("Kafka consumer initialized");
        } catch (Throwable t) {
            // ...
        }
    }

```

A lot of interesting stuff is going on here. We first see that all the major consumer properties are being initialized: `clientId`, `groupID`, `requestTimeoutMs` ( a single request timeout), `defaultApiTimeoutMs` (a client API operation timeout such as `Fetch`) and `deserializers` (plugins that turn key and value bytes in java objects).

After that, the building blocks of our consumer are initialized:
* `assignors`: to assign partition to consumers
* `subscriptions`: state of our subscriptions
* `coordinator`: group coordinator.
*  Fetchers: `fetcher`, `offsetFetcher` and `topicMetadataFetcher`.

## Conclusion


If you want to get in touch, you can find me on Twitter/X at [@moncef_abboud](https://x.com/moncef_abboud).  
