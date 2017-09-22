# Sandglass [![Build Status](https://img.shields.io/travis/celrenheit/sandglass.svg?style=flat-square)](https://travis-ci.org/celrenheit/sandglass) [![GoDoc](https://img.shields.io/badge/godoc-reference-5272B4.svg?style=flat-square)](https://godoc.org/github.com/celrenheit/sandglass) [![License](https://img.shields.io/badge/license-apache-blue.svg?style=flat-square)](LICENSE) [![Go Report Card](https://goreportcard.com/badge/github.com/celrenheit/sandglass?style=flat-square)](https://goreportcard.com/report/github.com/celrenheit/sandglass)

Sandglass is a distributed, horizontally scalable, persistent, delayed message queue. It was developped to support asynchronous tasks. It supports synchronous tasks as well. It supports the competing consumers pattern.

## Features

* Horizontal scalability
* Highly available
* Persistent storage
* Roughly strong ordering with a single consumer in a consumer group
* Round robin consumption between multiple consumers in a consumer group (looses ordering)
* Produce message to be consumed in the future
* Acknowledge each message individualy
* Automatic consumer offset tracking

## Project status

**EXPERIMENTAL**: This is a prototype. This should not be used in production in its current form.

See TODO section below for more information


## Architecture

### General

```
                          +--------------------------+
                          |                          |
                          |    Sandglass Cluster     |
                          |                          |
                          |                          |            +-----------------+
+-----------------+       |   +------------------+   |            |                 |
|                 |       |   |                  |   |     +------>  Consumer       |
|  Producer       +------->   |                  |   |     |      |                 |
|                 |       |   |    Broker 1      |   |     |      +-----------------+
+-----------------+       |   |                  |   |     |
                          |   |                  |   +-----+  Round robin consumption
                          |   +------------------+   |     |
+-----------------+       |                          |     |      +-----------------+
|                 |       |   +------------------+   |     |      |                 |
|  Producer       +------->   |                  |   |     +------>  Consumer       |
|                 |       |   |                  |   |            |                 |
+-----------------+       |   |    Broker 2      |   |            +-----------------+
                          |   |                  |   |
                          |   |                  |   |
+-----------------+       |   +------------------+   |            +-----------------+
|                 |       |                          |            |                 |
|  Producer       +------->   +------------------+   +------------+  Consumer       |
|                 |       |   |                  |   |            |                 |
+-----------------+       |   |                  |   |            +-----------------+
                          |   |    Broker 3      |   |
                          |   |                  |   |
                          |   |                  |   |
                          |   +------------------+   |
                          |                          |
                          |                          |
                          +------+----------^--------+
                                 |          |
                                 |          |
                  +--------------v----------+-----------------+
                  |                                           |
                  |          etcd, zookeeper, consul          |
                  |                                           |
                  +-------------------------------------------+
```


### Topics

There is two kinds of topics:
* Timer:
   * Fixed number of partitions (set up-front, could change)
   * Time ordered using [sandflake IDs](https://https://github.com/celrenheit/sandflake)
   * Can produce messages in the future

* Compacted (might change the name for this):
   * Fixed number of partitions (set up-front, cannot change)
   * Behaves like a distributed key value store


A topic has a number of partitions.
Data is written into a single partition. Either the destination partition is specified by the producer. Otherwise, we fallback to choosing the destination partition using a consistent hashing algorithm.

Each produced message to a partition writes a message to a Write Ahead Log (WAL) and to the message log.
The WAL is used for the replication logic, it is sorted in the order each message was produced.
The message log is used for message comsumption, it is mainly sorted by time (please refer to [sandflake ids](https://https://github.com/celrenheit/sandflake) for the exact composition)

The content of the message is stored in the message log and not in the WAL (only the keys are important). This way the message log is used for fast consumption avoiding random reads. 

This will probably change in order to have the WAL as the only source of truth instead of storing the content in the message log. This of course will have an impact because we are transfering random reads to the consumption path. Utlimately, we are going to have to store the content of the message in both logs for better performance at the cost of disk space.


A message is composed of the following fields:

        topic
        partition

        index   <- position in the WAL

        offset  <- position in the message log for timer topics
        key     <- position in the message log for key for compacted topics

        value   <- your payload


## Installation

As of now there is no binaries available, you can only install from source using:

```shell
$ go get -u github.com/celrenheit/sandglass/cmd/sandglass
```

## Usage

You need a running instance of [etcd](https://github.com/coreos/etcd).

All data will be stored in /tmp/node1 and /tmp/node2 for the second.

Open a first terminal window:

```shell
$ sandglass --config https://raw.githubusercontent.com/celrenheit/sandglass/master/demo/node1.yaml
```

On a second terminal window:

```shell
$ sandglass --config https://raw.githubusercontent.com/celrenheit/sandglass/master/demo/node2.yaml
```

## TODO

* Clean up all the mess
* Fix replication and re assign partitions correctly when a node goes down
* Save all the registered nodes and not rely on gossip to allow topic creation even if there is not enough nodes
* More TODOs in TODO section (#inception)
* Make everything more robust...