---
layout: page
title: Floodplain: Running the demo
subtitle: Running the demo setup
image: /img/river.jpg
---

## All-in-one demo setup

If you want to play with the demos, we strongly recommend just starting with the [all-in-one repository](https://github.com/floodplainio/floodplain-demo-setup).

This is a docker-compose file that creates a Postgres database, the movie-rental dataset, an application that modifies the dataset periodically, a kafka cluster, a kafka manager ui and a mongodb sink.

The readme explains in more detail.

When this set of services are running, you can run the [examples](https://github.com/floodplainio/floodplain-library/tree/master/floodplain-example/src/main/kotlin/io/floodplain/kotlindsl/example)
