---
layout: page
title: Floodplain Deepdive
subtitle: How does Floodplain work?
image: /img/river.jpg
---

## Deep dive

So how does Floodplain work?

```kotlin
fun main() {
    stream("mygeneration") {
        val pgConfig = postgresSourceConfig("mypostgres","postgres",5432,"postgres","mysecretpassword","dvdrental")
        val mongoConfig = mongoConfig("mymongo","mongodb://mongo","mydatabase")
        postgresSource("public","actor",pgConfig) {
            mongoSink(topologyContext,"mycollection","sometopic",mongoConfig)
        }
    }.renderAndStart(URL( "http://localhost:8083/connectors"),"kafka:9092")
}
```

When we run this code, the stream() function call creates a stream object, this is the toplevel floodplain construct. The stream contains one or more (toplevel) source objects. Every source object contains zero or more transformations, terminated by a sink.
Some transformations, like joins, can contain other sources.

The streams() call takes a few optional parameters (more on that later), and one targeted lambda expression and returns a Stream() object.
On the Stream object we can call the render() method, which returns three values (using a Triple): A list of source configurations, a list of sink configurations and a Kafka Streams Topology.

The source and sink configurations are JSON strings, which we can POST to a Kafka Connect instance, and finally we can use a KafkaStreams instance to run the Topology instance.

We can shorthand this by calling this method on stream:

```kotlin
fun renderAndStart(connectorURL:URL, kafkaHosts: String) {}
```

... where we have to supply the URL to post the JSON config objects, a connection string for the Kafka cluster, and finally a clientId for the streams run.

## Generations

Previously we mentioned the 'generation' string when rendering a stream. We use that string to differentiate different runs of a topology. We need something like this due to the stateful nature of Floodplain runs. Remember, when we start a run, it will start reading the sources, perform the transformations and send the results to the sinks. It will create internal topics to perform some transformations, and all stateful transformations are backed by state stores, which are also stored in KAfka.
Now if we stop this instance, and start it again, it will continue where it left off.
If we want to change something in the code, we can do so, and restart again, but then we end up in a weird state: Everything up to now, all stateful tranformations and sinks contain data created by the 'old' code, but every new change will be processed by the new code. This might create a result that neither the old code nor the new code could create, so changing a 'running' topology (and by running I include stopped and later continued instances) should be done with great care.

Usually it is wiser to create a new generation: Use different topics, different consumerIds, so it starts from scratch. An additional benefit of having an entire new set of consumers and topics, is that we can leave the old topology running, and running the new one in parallel. When both run in parallel, we can assess both topologies, and if we are happy with the new version, and we're sure the old version is no longer used, we can stop and delete the old instance.

This is especially important when we are running large data sets: If we are running floodplain on a table of 100M records, getting a new generation up and running can take hours, and we want to upgrade without downtime.

## Threading

TODO

## Supported sources and sinks

For sources, we support debezium now, and for configuration we've only implemented Postgres. Other debezium sources (MySQL, SQLServer, MongoDB, Cassandra (experimental) and Oracle (experimental) should be pretty trivial to implement, as the configuration and data should be compatible. The biggest hurdle for writing an implementation is needing a good test data set, contributions are welcome.

For sinks we support MongoDB, HTTP, and Google Sheets (experimental). In theory, adding sources or sinks should be trivial, but in practice there are always some surpises that need adressing.

The postgres source:

```json
{
  "name": "mytenant-mydeployment-mypostgres",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.dbname": "dvdrental",
    "database.user": "postgres",
    "database.password": "mysecretpassword",
    "name": "mytenant-mydeployment-mypostgres",
    "database.server.name": "mytenant-mydeployment-mypostgres"
  },
  "tasks": [],
  "type": "source"
}
```

The mongodb sink:

```json
{
  "name": "mytenant-mydeployment-mymongo",
  "config": {
    "connector.class": "com.mongodb.kafka.connect.MongoSinkConnector",
    "value.converter.schemas.enable": "false",
    "key.converter.schemas.enable": "false",
    "value.converter": "com.dexels.kafka.converter.ReplicationMessageConverter",
    "key.converter": "com.dexels.kafka.converter.ReplicationMessageConverter",
    "document.id.strategy": "com.mongodb.kafka.connect.sink.processor.id.strategy.FullKeyStrategy",
    "connection.uri": "mongodb://mongo",
    "database": "mytenant-mydeployment-mygeneration-myinstance-mydatabase",
    "collection": "mycollection",
    "topics": "mytenant-mydeployment-sometopic",
    "topic.override.mytenant-mydeployment-sometopic.collection": "mycollection",
    "name": "mytenant-mydeployment-mymongo",
    "database.server.name": "mytenant-mydeployment-mymongo"
  },
  "tasks": [],
  "type": "sink"
}
```

It will tell the postgres source and mongodb sink all it needs to know to get the data from the source into kafka and from kafka to the sink.

Finally, we have the transformation. We create a so called 'topology', which is a standard Kafka Streams construct (using the Processor API) which we can then run. A topology is a sequence (actually a directed graph) of sources, processors and sinks. If we ask Kafka Streams to describe it, it looks like this:

```
Topology:
 Topologies:
   Sub-topology: 0
    Source: mytenant-mydeployment-mypostgres.public.actor (topics: [mytenant-mydeployment-mypostgres.public.actor])
      --> mytenant-mydeployment-myFtion-myinstance-debezium_debconv_1_0
    Processor: mytenant-mydeployment-mygeneration-myinstance-debezium_debconv_1_0 (stores: [])
      --> mytenant-mydeployment-mygeneration-myinstance-debezium_deb_1_0
      <-- mytenant-mydeployment-mypostgres.public.actor
    Processor: mytenant-mydeployment-mygeneration-myinstance-debezium_deb_1_0 (stores: [])
      --> mytenant-mydeployment-mygeneration-myinstance-set_1_1
      <-- mytenant-mydeployment-mygeneration-myinstance-debezium_debconv_1_0
    Processor: mytenant-mydeployment-mygeneration-myinstance-set_1_1 (stores: [])
      --> mytenant-mydeployment-mygeneration-myinstance-filter_1_2
      <-- mytenant-mydeployment-mygeneration-myinstance-debezium_deb_1_0
    Processor: mytenant-mydeployment-mygeneration-myinstance-filter_1_2 (stores: [])
      --> SINK_mytenant-mydeployment-sometopic
      <-- mytenant-mydeployment-mygeneration-myinstance-set_1_1
    Sink: SINK_mytenant-mydeployment-sometopic (topic: mytenant-mydeployment-sometopic)
      <-- mytenant-mydeployment-mygeneration-myinstance-filter_1_2
```

Not the most helpful format, it gets better with some visualization (kudos to zz85 to create [this open source visualization](https://zz85.github.io/kafka-streams-viz/), it's a tremendous help when figuring out topology issues )

![alt text](/img/topology.png "Topology image")

A more complex one:

![alt text](/img/canvas.png "Topology image")
