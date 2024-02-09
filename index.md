---
title: Home
layout: home
nav_order: 1
---

Titanis is an open-source library for Apache Spark that streamlines concept drift detection for machine learning pipelines. In addition to drift detection, Titanis comprises incremental machine learning algorithms optimized for online learning and distributed systems. Titanis is built on Apache Spark and supports the Spark Structured Streaming API, making it easy to integrate into existing Spark Streaming pipelines.

Titanis is built using Scala, which runs on the Java Virtual Machine. This not only enables native implementations of drift detection and machine learning algorithms to work with Spark Streaming, but also supports the usage of Java-implemented drift detectors and learners from the extensive library of MOA. This gives users of Titanis access to the wide range of algorithms already in place in MOA and makes it simpler to integrate MOA algorithms in existing Spark pipelines. The learning algorithms in Titanis are specially designed to work for streaming data in a distributed fashion using Apache Spark. Titanis trains, evaluates, and tests ML models and ensembles in a distributed manner, leveraging the fault-tolerant and scalable Apache Spark Structured Streaming engine.
