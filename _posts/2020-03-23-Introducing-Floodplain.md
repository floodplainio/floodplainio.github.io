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

## Why? What problem does it solve?

Let's take an example.

We have an application, that uses a database. If you want to play along, clone this repository:

https://github.com/floodplainio/floodplain-cdc.git

This is a Kotlin + Quarkus application, as well as a Postgres database.
There is already a datamodel present, and the application will periodically insert random payments into the payment table.

It is pretty easy to get started, just follow the README in the repository:
https://github.com/floodplainio/floodplain-cdc/blob/master/README.md

So now we have an application running, with actual data, and actual changes being made to the database.
