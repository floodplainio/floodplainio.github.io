---
layout: page
title: Floodplain: Running the standalone demo app
subtitle: Running the demo setup
image: /img/river.jpg
---

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
