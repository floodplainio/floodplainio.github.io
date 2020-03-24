---
layout: post
title: Introducing Floodplain
subtitle: What is Floodplain?
image: /img/hello_world.jpeg
---
Floodplain is a data liberation platform!
In a nutshell, you point it to a database, floodplain will receive all changes to that database (even logically speaking all the changes that have occurred up to that point), then it will optionally transform those changes in some way, and finally it will push those changes to another database.

So that gives us a new, near-realtime updated materialized view in a new database.

## Why? What problem does it solve?
Let's take an example.

We have an application, that uses a database. If you want to play along, clone this repository:

https://github.com/floodplainio/floodplain-cdc.git

This is a Kotlin + Quarkus application, as well as a Postgres database.
There is already a datamodel present, and the application will periodically insert random payments into the payment table.

It is pretty easy to get started, just follow the README in the repository: 
https://github.com/floodplainio/floodplain-cdc/blob/master/README.md

So now we have an application running, with actual data, and actual changes being made to the database.


