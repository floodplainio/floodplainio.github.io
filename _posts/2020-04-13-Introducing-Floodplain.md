---
layout: page
title: Introducing Floodplain
subtitle: What is Floodplain?
image: /img/river.jpg
---

# Floodplain

Floodplain is a data liberation platform!
In a nutshell, you point it to a database, floodplain will receive all changes to that database (even logically speaking all the changes that have occurred up to that point), then it will optionally transform those changes in some way, and finally it will push those changes to another database.

So that gives us a new, near-realtime updated materialized view in a new database.

It's closest competitor would be KSQLdb. Where KSQLdb implements a SQL variant, we chose for a more general purpose language. Which works better for you will obviously depend on use case and personal preference.

First of all, KSQL(db) is much more mature, and Floodplain is very much alpha. KSQLdb is developed by Confluent, the company behind Kafka, where Floodplain is develped as an unpaid open source project by a tiny team.

So while not nearly as mature as KSQL, there are reasons to favor Floodplain's approach though:

### SQL

Perhaps personal taste, but I don't like SQL much. It has served the world well, and there are many very skilled people who can do amazing things with it, that does not necessarily translate to being the best choice for essentially a new field.

The streaming SQL version of KSQL is another extension of an already pretty weak standard, while different versions of SQL seem similar, switching between implementations is far from painless, and the superficial similarities actually make it harder.

and we think that embracing a new language seems more effective than forcing a query language further and further into a non-query domain. Also static analysis of SQL is tricky, as is IDE integration.

KSQL is working on improving this, integrating existing code into Floodplain streams is much easier. All stateful and stateless transformations are just Kotlin and Java code, so it is much more straightforward to integrate.

[Using Floodplain](/2020-04-12-Using-Floodplain/)
