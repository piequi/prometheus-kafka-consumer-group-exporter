Prometheus Kafka Consumer Group Exporter
====
This Prometheus exporter consumes the `__consumer_offsets` topic of a Kafka cluser and exports the results as Prometheus gauge metrics. i.e. it shows the position of Kafka consumer groups, including their lag.

The high-water and low-water marks of the partitions of each topic are also exported.

# Packaging
In order to install this exporter on offline servers (without internet access), you first need to package pip dependencies.

To download required python modules for offline pip installation, run:
```
mkdir pip-dependencies
pip3 download kafka-python==1.3.5 jog prometheus-client javaproperties -d "pip-dependencies"
```

Then, add the `pip-dependencies` directory to the TAR file containing the exporter.

# Usage
Once installed, you can run the exporter with :
```
/usr/bin/python3 -m prometheus_kafka_consumer_group_exporter
```
> You will have to tell python where to find the module `prometheus_kafka_consumer_group_exporter` using the environment variable `PYTHONPATH`. Simply add the path of the directory where you've installed the exporter in that variable.

By default, it will bind to port 9208 and connect to Kafka on `localhost:9092`. You can change these defaults as required by passing in arguments:
```
prometheus_kafka_consumer_group_exporter -p <exporter port> --consumer-config <config file path>
```
Run with the `-h` flag to see details on all the available arguments.

Prometheus metrics can then be scraped from the `/metrics` path, e.g. http://localhost:9208/metrics. Metrics are currently actually exposed on all paths, but this may change in the future and `/metrics` is the standard path for Prometheus metric endpoints.

# Metrics
Nine main metrics are exported:

### `kafka_consumer_group_offset{group, topic, partition}`
The latest committed offset of a consumer group in a given partition of a topic, as read from `__consumer_offsets`. Useful for calculating the consumption rate and lag of a consumer group.

### `kafka_consumer_group_lag{group, topic, partition}`
The lag of a consumer group behind the head of a given partition of a topic - the difference between `kafka_topic_highwater` and `kafka_consumer_group_offset`. Useful for checking if a consumer group is keeping up with a topic.

### `kafka_consumer_group_lead{group, topic, partition}`
The lead of a consumer group ahead of the tail of a given partition of a topic - the difference between `kafka_consumer_group_offset` and `kafka_topic_lowwater`. Useful for checking if a consumer group is at risk of missing messages due to the cleaner.

### `kafka_consumer_group_commits{group, topic, partition}`
The number of commit messages read from `__consumer_offsets` by the exporter from a consumer group for a given partition of a topic. Useful for calculating the commit rate of a consumer group (i.e. are the consumers working).

### `kafka_consumer_group_exporter_offset{partition}`
The offset of the exporter's consumer in each partition of the `__consumer_offset` topic. Useful for calculating the lag of the exporter.

### `kafka_consumer_group_exporter_lag{partition}`
The lag of the exporter's consumer behind the head of each partition of the `__consumer_offset` topic. Useful for checking if the exporter is keeping up with `__consumer_offset`.

### `kafka_consumer_group_exporter_lead{partition}`
The lead of the exporter's consumer ahead of the tail of each partition of the `__consumer_offset` topic. Useful for checking if the exporter is at risk of missing messages due to the cleaner.

### `kafka_topic_highwater{topic, partition}`
The offset of the head of a given partition of a topic, as reported by the lead broker for the partition. Useful for calculating the production rate of the producers for a topic, and the lag of a consumer group (or the exporter itself).

### `kafka_topic_lowwater{topic, partition}`
The offset of the tail of a given partition of a topic, as reported by the lead broker for the partition. Useful for calculating the lead of a consumer group (or the exporter itself) - i.e. how far ahead of the cleaner the consumer group is.

## Lag
Lag metrics are exported for convenience, but they can also be calculated using other metrics if desired:
```
# Lag for a consumer group:
kafka_topic_highwater - on (topic, partition) kafka_consumer_group_offset{group="some-consumer-group"}

# Lag for the exporter:
kafka_topic_highwater{topic='__consumer_offsets'} - on (partition) kafka_consumer_group_exporter_offset
```
Note that as the offset and high-water metrics are updated separately the offset value can be more up-to-date than the high-water, resulting in a negative lag. This is often the case with the exporter lag, as the exporter offset is tracked internally rather than read from `__consumer_offsets`.

# Kafka Config
If you need to set Kafka consumer configuration that isn't supported by command line arguments, you can provided a standard Kafka consumer properties file:
```
prometheus_kafka_consumer_group_exporter --consumer-config consumer.properties
```
See the [Kafka docs](https://kafka.apache.org/documentation/#newconsumerconfigs) for details on consumer properties. However, as the exporter doesn't use the official consumer implementation, all properties may not be supported. Check the [kafka-python docs](https://kafka-python.readthedocs.io/en/master/apidoc/KafkaConsumer.html#kafkaconsumer) if you run into problems.

You can provide multiple files if that's helpful - they will be merged together, with later files taking precedence:
```
prometheus-kafka-consumer-group-exporter --consumer-config consumer.properties --consumer-config another-consumer.properties
```
Note that where a command line argument relates to a consumer property (e.g. `--bootstrap-brokers` sets `bootstrap.servers`) a value provided via that argument will override any value for that property in a properties file. The argument default will only be used if the property isn't provided in either a file or an argument.
