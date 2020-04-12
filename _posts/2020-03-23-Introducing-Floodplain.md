---
layout: post
title: Introducing Floodplain
subtitle: What is Floodplain?
image: /img/hello_world.jpeg
---

Floodplain is a data liberation platform!
In a nutshell, you point it to a database, floodplain will receive all changes to that database (even logically speaking all the changes that have occurred up to that point), then it will optionally transform those changes in some way, and finally it will push those changes to another database.

So that gives us a new, near-realtime updated materialized view in a new database.

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

https://kotlinlang.org/docs/reference/lambdas.html#function-literals-with-receiver

After that

## Example

Let's take an example.

We have an application, that uses a database. If you want to play along, clone this repository:

https://github.com/floodplainio/floodplain-cdc.git

This is a Kotlin + Quarkus application, as well as a Postgres database.
There is already a datamodel present, and the application will periodically insert random payments into the payment table.

It is pretty easy to get started, just follow the README in the repository:
https://github.com/floodplainio/floodplain-cdc/blob/master/README.md

So now we have an application running, with actual data, and actual changes being made to the database.
