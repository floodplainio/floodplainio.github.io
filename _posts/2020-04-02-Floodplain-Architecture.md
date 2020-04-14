---
layout: page
title: Floodplain Architecture
subtitle: How do I run Floodplain?
image: /img/river.jpg
---

Floodplain is meant to run inside a container, and probably under some kind of container orchestrator like Kubernetes.

Still, running Floodplain is pretty easy. It is a regular JVM program.

A thing to keep in mind: Floodplain does not run any Kafka Connectors, it just creates configuration objects for a Kafka Connect instance that needs to contain all the required connectors and dependencies.

We're still unsure how to build that effectively. At the moment we're still deciding if we would prefer multiple connector instances (So we can use off-the-shelf container instances of our connectors), or try to build a 'super connector' that contains all required dependencies.

TBD, I guess.

TODO note: Currently researching Quarkus, possibly combined with GraalVM native.
