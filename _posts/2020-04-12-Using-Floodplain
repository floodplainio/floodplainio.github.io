---
layout: page
title: Using Floodplain
subtitle: How to use Floodplain?
image: /img/river.jpg
---
## How does it work?

Floodplain is a Kotlin library. Most of it is written in Java, but the fact that Kotlin has extension functions and much better DSL capabilities, we chose for Kotlin.

We can write code in a Kotlin DSL that defines how our system can acquire data, transform it, and send it to a sink.

This Kotlin DSL is just a certain 'shape' we put our code in, for the rest it is just regular Kotlin, so we can use all kind of 3rd party Kotlin Libraries (or other JVM libraries for that matter)

We will 'compile' that code to a Kafka Streams definition (a 'topology' as we call it in the Kafka lingo), and a set of 'connect configs' that we can send to a Kafka Connect instance, so it will know how to configure the data sources and sinks.

So in pseudo code it looks something like this
(define source) -> (define transformation) -> (define sink)

To make it a bit more specific:
(configure sources and sinks)
(define source) -> (define transformation) -> (define sink)
... repeat as necessary

In real code it does get a bit more involved. Let's take a concrete example where we want to stream the contents of one single table in Postgres into a collection in MongoDB, without any fancy transformations in between.

It looks something like this:

```kotlin
fun main() {
    pipe("mygeneration") {
        val pgConfig = postgresSourceConfig("mypostgres","postgres",5432,"postgres","mysecretpassword","dvdrental")
        val mongoConfig = mongoConfig("mymongo","mongodb://mongo","mydatabase")
        postgresSource("public","actor",pgConfig) {
            mongoSink(topologyContext,"mycollection","sometopic",mongoConfig)
        }
    }.renderAndStart(URL( "http://localhost:8083/connectors"),"kafka:9092", UUID.randomUUID().toString())
}
```

Lets' go through this small program. First we create a configuration two configuration objects for Postgres and MongoDB. The functions that create these are specific for that endpoint, so it will be obvious which and what kind of parameters are needed for configuration. The function signature is:

```kotlin
fun Pipe.postgresSourceConfig(name: String, hostname: String,port: Int, username: String, password: String, database: String): PostgresConfig {}
```

This is an advantage of configuring a system using a strongly typed language. It is harder to get wrong, and an IDE can help you much better than when you are crafting a specific YAML to make ti work. Also, as we are in a regular Kotlin main function, we can supply these configuration properties in any way that works for us. In this case, we hard coded them in the code, that will usually not be optimal, but we can easily read them from environment variables, configuration files or some other configuration data source.

But the next part is where it gets interesting:

```kotlin
postgresSource("public","actor",pgConfig) {
	mongoSink("mycollection","sometopic",mongoConfig)
}
```

In this part we request the 'actor' table (in the 'public' schema) in our postgres config to be streamed though our system. After that we stream it directly to a mongodb sink, to a collection "mycollection" (using a topic called "sometopic" but that does not matter too much here)

Note for those unfamiliar with the Kotlin lambda syntax, the postgresSource is a method call with 4(!) parameters, two strings, a configuration object and a lambda. So the part between the curly brackets is actually the last parameter.

That lambda is a so called lambda with a receiver, so aside from supplying a function, we also supply a receiver, basically what 'this' means in the context of that function.

This is the method signature of the 'postgresSource' function:

```kotlin
fun Pipe.postgresSource(schema: String, table: String, config: PostgresConfig, init: Source.() -> Unit): Source {}
```

So whats happening here: The postgresSource method creates a Source object, and runs the lambda in the context of that source object. So within that lambda we can behave as if we are directly defining a method to the Source object.

Perhaps this is obvious to Kotlin veterans, but it is essential to understand this aspect.

[kotlin reference](https://kotlinlang.org/docs/reference/lambdas.html#function-literals-with-receiver)

Now, back to our example. In this case, the only thing we supply is a sink transformation. Generally all source instances end with a sink (or possibly more than one (Not implemented at the moment))

Let's make our example a little bit more interesting.

```kotlin
postgresSource("public","actor",pgConfig) {
	set { msg,_ -> msg["last_update"]=null; msg}
	filter {msg,_ ->(msg["actor_id"] as Int) < 10}
	mongoSink("mycollection","sometopic",mongoConfig)
}
```

We've added two transformers to our source. First, 'set' is a single message transformation. So every message gets passed through this function, and will return a single message as well (Not completely true: In a later chapter we'll address the second parameter, usually called state). In this case, we'll remove a column, the 'last_update' column. This is a field that was in the original Postgres database, and if we know we're not interested in this field downstream, it makes sense to remove it as soon as possible.

The second transformation is a filter. A filter wants a lambda that takes a message, and returns a boolean. If true, the message will propagat, otherwise the message gets dropped. In this case we take the actor_id field, assert that it is an integer, and only propagate actor_id that is lower than 10.

After that we still have the same mongodb sink, that will collect the data into the mongodb collection.

## Deep dive

Depending on your learning style, you might want to look into examples first, otherwise we'll explain what happens under the hood.
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
      --> mytenant-mydeployment-mygeneration-myinstance-debezium_debconv_1_0
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

## Stateful transformations

In the original example we did a few transformations, but only transformations that involve only one message. The filter transformation decides to let a message pass or not, and the 'set' operation changes every message in the same way.

That does not cover all cases. Sometimes we need transformations that 'remember' messages that have come before.

A very common example of one of these operations is a join: Imagine we have two tables, we create two sources to receive those tables, and we'd like to join the records of these tables to one, combined record. For simplicity's sake, lets assume these tables share the same key space.

Now one thing we can't do (at least at this point) is _querying_ data. We receive versions of records. An insert is a record with a new, unseen key. An update is a new version of a record with a known key. A delete is a known key, but with no record at all. We have no control of when we get those updates.
So if we want to join two topics, we need to store the records of both topics. When a record appears on topic A, we check our local store of topic B, to see if there is a matching record with the same key. If we find one, we combine the two (somehow). If there is no matching record in Topic B, we simply save the message in the local storage of Topic A. When, at some point, a record appears in topic B with the matching key, we will retrieve the record from topic A.

This is pretty straightforward. In floodplain Kotlin DSL this would look like this:

```kotlin
    pipe("generation") {
        val pgConfig = postgresSourceConfig("mypostgres","postgres",5432,"postgres","mysecretpassword","dvdrental")
        val mongoConfig = mongoConfig("mymongo","mongodb://mongo","mydatabase")
        postgresSource("public","tableA",pgConfig) {
            joinWith {
                postgresSource("public","tableB",pgConfig) {}
            }
            set {
                msg,msg2->msg["topicBSubMessage"]=msg2; msg
            }
            mongoSink("mycollection","sometopic",mongoConfig)
        }
    }
```

We add a joinWith transformer to our source (the topicA source), which will contain another source, topicB
After this transformation, whenever a join matches we _still have two messages_, one from topicA, and one from topicB, both with the same key.
Every transformer can emit one 'main' message, and another secondary message. The semantics of the second message depends on the transformation. In the case of a join, it is the message the main source was joined with.

Secondary messages don't survive, they only make it to the next transformer, so if you want to save anything, you will need to merge the messages. In this case, in the 'set' operator, we add the entire secondary message as a submessage into the main message.

This is pretty easy to express but sadly, this particular situation is actually pretty uncommon in SQL databases (meaning two tables that share the same key space). Usually it gets a bit more complex, for example when there is a one-to-many relationship between tables.

Let's take a look at the (simplified) data model of our example app. Two tables, film and language. Every film has a field 'language_id', which points to a record in the language. In PostgresDDL, the Film table:

```sql
CREATE TABLE
    film
    (
        film_id SERIAL NOT NULL,
        title CHARACTER VARYING(255) NOT NULL,
        language_id SMALLINT NOT NULL,
        CONSTRAINT film_language_id_fkey FOREIGN KEY (language_id) REFERENCES "language"
        ("language_id") ON
    );
```

and the language table:

```sql
CREATE TABLE
    language
    (

        language_id SERIAL NOT NULL,
        name CHARACTER(20) NOT NULL,
        PRIMARY KEY (language_id)
    );
```

So if I want to include the name of the language in a downstream film, I need to join these two. Clearly in this case the two tables have different keys (film_id vs. language_id). Even if they would have the same name and type, they are different keys: Film # 1 is something completely different to Language # 1.
To make this work we will need to access the language_id field of film, and use that to join with the language table. For that, we have the joinRemote transformation. In a join remote transformation we have an additional parameter, a lambda that extracts the remote key from the message.

Let's have a look at another example:

```kotlin
postgresSource("public","film",pgConfig) {
	joinRemote({msg->msg["language_id"].toString()}) {
		postgresSource("public","language",pgConfig) {} }
	set {
		film,language->film["language"]=language["name"]; film
	}
	mongoSink("filmwithlanguage","filmwithlanguage",mongoConfig)
}
```

The key extraction lambda:

```kotlin
{msg->msg["language_id"].toString()}
```

Will extract the language_id field from the message, and convert it to a string (in floodplain **all keys** are strings.)
After that, the set statement will add the 'name' field from the language record to the film record, and send it on its way to mongodb.
There are some interesting observations to make here: This is a many-to-one relationship: Many films exist that share the same language, while (in this data model at least) films have only one language.
So once this floodplain transformation is running, every time a film changes, this join is performed again, and the new record is inserted into mongodb.
However, if a language name changes (I know, does not seem too common, but for arguments sake), that would imply that every film in that language should be re-joined with that new name, and sent to mongodb again.
So if my database has 1000 movies in Spanish, and I would like to change the name of the Spanish language to 'Castillian', that will result in 1000 new inserts into Mongodb.
It is important to be aware of these 'write magnification' effects, because as topologies get bigger, tiny source changes can have a 'butterfly effect' in the number of downstream changes.

So after this, in the resulting MongoDB collection, all records will have a 'language' field, that will show the string representation of the language. If the source database language changes, all MongoDB records will be updated.

In a situation like this it might also make sense to remove the language_id field, as it is pretty meaningless in that space.

## Aggregations
TODO

## Splitting topics
TODO

## Bufferering



## Example

Let's take an example.

We have an application, that uses a database. If you want to play along, clone this repository:

https://github.com/floodplainio/floodplain-cdc.git

This is a Kotlin + Quarkus application, as well as a Postgres database.
There is already a datamodel present, and the application will periodically insert random payments into the payment table.

It is pretty easy to get started, just follow the README in the repository:
https://github.com/floodplainio/floodplain-cdc/blob/master/README.md

So now we have an application running, with actual data, and actual changes being made to the database.

As stated in the readme, you can connect to this database using this connection string:

```
postgresql://localhost:mysecretpassword@localhost:5432/dvdrental
```
