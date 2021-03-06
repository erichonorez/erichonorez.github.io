## Configuration

### Producer

```
bootstrap.servers=broker1:9091,broker2:9091
```

**Required** A list of host/port pairs to use for establishing the initial connection to the Kafka cluster. Then the producer will discover the all kafka cluster so it is not necessary to put the full list of kafka brokers in this property. Put just one more in case of failure.

```
acks=all
```

**Optional** The number of acknowledgments the producer requires the leader to have received before considering a request complete. *acks=all* means the leader will wait for the full set of in-sync replicas to acknowledge the record. This guarantees that the record will not be lost as long as at least one in-sync replica remains alive. Default value is *1* meaning the leader will write the record to its local log but will respond without awaiting full acknowledgement from all followers.

```
client.id="my-application"
```

**Optional** An id string to pass to the server when making requests. Allowing a logical application name to be included in server-side request logging. If there are several instances of the same application it may be a good idea to append the instance id after the application name. E.g. "my-application-1" or "my-application-2".

```
enable.idempotence=true
```

**Optional** Enables the exactly-once delivery semantic. Default is *false*

```
transactional.id=my-transaction-id-1
```

**Optional** Enables transactions by specifying a transactional identifier that will be use by the producer. Note that **enable.idempotence** must be enabled if a *TransactionalId* is configured. Default value is an empty string.

#### Summary

```java
Properties props =  new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
props.put(ProducerConfig.CLIENT_ID_CONFIG, "kafka-app-test");
props.put(ProducerConfig.ACKS_CONFIG, "all");
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "kafka-app-test-producer-1");
return props;
```

#### Resources

* [Apache Kafka documentation](http://kafka.apache.org/documentation/#producerconfigs)

### Consumer

```
bootstrap.servers=broker1:9091,broker2:9091
```

**Required** A list of host/port pairs to use for establishing the initial connection to the Kafka cluster. The client will make use of all servers irrespective of which servers are specified here for bootstrapping—this list only impacts the initial hosts used to discover the full set of servers.


```
group.id="my-application"
```

**Required** A unique string that identifies the consumer group this consumer belongs to. The same message will be delivered exactly once to the same *group.id*. 

```
client.id="my-application"
```

An id string to pass to the server when making requests. Allowing a logical application name to be included in server-side request logging. If there are several instances of the same application it may be a good idea to append the instance id after the application name. E.g. "my-application-1" or "my-application-2".

```
auto.offset.reset=earliest
```

**Optional** What to do when there is no initial offset in Kafka or if the current offset does not exist any more on the server. *earliest* will reset the set at the beginning. Other option is *lastest* which set the offset at the last one. Use *earliest* if you want your application processes all messages published even before it being up. Use *latest* if you want your application only process new messages. The default value is *latest*

```
isolation.level=read_committed
```

**Optional** Specify that the consumer just consumes the committed messages or messages that are not part of a transaction. The default value is *read_uncommitted* which means that `consumer.poll()` method returns all messages even if the transaction has been aborted.

```
enable.auto.commit=false
```
**Optional** If true the consumer's offset will be periodically committed in the background. Set it to *false* if you want create transactional consumers.

#### Summary

```java
Properties props = new Properties();
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ConsumerConfig.GROUP_ID_CONFIG, "kafka-app-test-consumer");
props.put(ConsumerConfig.CLIENT_ID_CONFIG, "kafka-app-test-consumer");
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
return props;
```

#### Resources

* [Apache Kafka documentation](http://kafka.apache.org/documentation/#consumerconfigs)

## Transactions

All of the following snippets of text and code come from [Transactions in Apache Kafka](https://www.confluent.io/blog/transactions-apache-kafka/)

*"Transactions enable atomic writes to multiple Kafka topics and partitions. All of the messages included in the transaction will be successfully written or none of them will be. For example, an error during processing can cause a transaction to be aborted, in which case none of the messages from the transaction will be readable by consumer."*

*"A message is considered consumed only when its offset is committed to the offsets topic."*

*"The Kafka consumer will only deliver transactional messages to the application if the transaction was actually committed. Put another way, the consumer will not deliver transactional messages which are part of an open transaction, and nor will it deliver messages which are part of an aborted transaction." … "In short: Kafka guarantees that a consumer will eventually deliver only non-transactional messages or committed transactional messages. It will withhold messages from open transactions and filter out messages from aborted transactions."*

### Example

In the following example a consumer consumes messages from an input topic and publish them to an output topic. Publishing to the output topic and committing the offsets of the consumer are in the same transaction and so in the same atomic write.

```java
KafkaProducer producer = createKafkaProducer(
  // Required producer config
  “bootstrap.servers”, “localhost:9092”,
  “transactional.id”, “my-transactional-id”);

producer.initTransactions(); // initialise the transactions for the given producer. Should only be executed once per producer.

KafkaConsumer consumer = createKafkaConsumer(
  // Required consumer config
  “bootstrap.servers”, “localhost:9092”,
  “group.id”, “my-group-id”,
  "isolation.level", "read_committed"); // the consumer only deliver messages of committed transactions or non transactional messages

consumer.subscribe(singleton(“inputTopic”));

while (true) {
  ConsumerRecords records = consumer.poll(Long.MAX_VALUE);
  producer.beginTransaction();
  for (ConsumerRecord record : records)
    producer.send(producerRecord(“outputTopic”, record));
  producer.sendOffsetsToTransaction(currentOffsets(consumer), group); // publish the offsets of the consumer
  producer.commitTransaction();
}
```

### Resources

* [Transactions in Apache Kafka](https://www.confluent.io/blog/transactions-apache-kafka/)
* [Transactional messaging in kafka](https://cwiki.apache.org/confluence/display/KAFKA/Transactional+Messaging+in+Kafka)

## Command line utilities

```bash
$ ./kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group kafkaspring-consumer
```

Describes consumer group and list offset lag (number of messages not yet processed) related to given group.

```bash
$ ./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic my-topic --from-beginning
```

Reads the messages on a topic from the beginning.

```bash
$ ./kafka-console-producer.sh --broker-list localhost:9092 --topic my-topic
```

Publishes messages on the topic *my-topic*.
