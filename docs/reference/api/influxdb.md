---
title: InfluxDB
description: InfluxDB line protocol reference documentation.
---

QuestDB implements the
[InfluxDB line protocol](https://docs.influxdata.com/influxdb/v1.8/write_protocols/line_protocol_tutorial/)
to ingest data. This enables you to use QuestDB as a drop-in replacement for
InfluxDB and others implementing the protocol. QuestDB can listen for line
protocol packets both over [TCP](#tcp-receiver) and [UDP](#udp-receiver).

:::info

This page describes InfluxDB line protocol APIs. More details on the protocol,
including including practical hints for understanding and working with the
message format can be found in the
[InfluxDB line protocol guide](/docs/guides/influxdb-line-protocol).

:::

## Usage

### Syntax

```shell
table_name,tagset fieldset timestamp
```

| Element      | Definition                                                                                                                          |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| `table_name` | Name of the table where QuestDB will write data.                                                                                    |
| `tagset`     | Array of string key-value pairs separated by commas that represent the reading's associated metadata                                |
| `fieldset`   | Array of key-value pairs separated by commas that represent readings. The keys are string, values can be string, numeric or boolean |
| `timestamp`  | UNIX timestamp. By default in nanoseconds. Can be changed in the configuration                                                      |

### Behaviour

- When the `table_name` does not correspond to an existing table, QuestDB will
  create the table on the fly using the name provided. Column types will be
  automatically recognized and assigned based on the data.
- The `timestamp` column is automatically created as
  [designated timestamp](/docs/concept/designated-timestamp/) with the
  [partition strategy](/docs/concept/partitions/) set to `NONE`. If you would
  like to define a partition strategy, you should
  [CREATE](/docs/reference/sql/create-table/) the table beforehand.
- When the timestamp is empty, QuestDB will use the server timestamp.

### Generic example

Let's assume the following data:

| timestamp           | city    | temperature | humidity | make      |
| ------------------- | ------- | ----------- | -------- | --------- |
| 1465839830100400000 | London  | 23.5        | 0.343    | Omron     |
| 1465839830100600000 | Bristol | 23.2        | 0.443    | Honeywell |
| 1465839830100700000 | London  | 23.6        | 0.358    | Omron     |

The line protocol syntax for that table is:

```shell
readings,city=London,make=Omron temperature=23.5,humidity=0.343 1465839830100400000
readings,city=Bristol,make=Honeywell temperature=23.2,humidity=0.443 1465839830100600000
readings,city=London,make=Omron temperature=23.6,humidity=0.348 1465839830100700000
```

### Irregularly-structured data

InfluxDB line protocol makes it possible to send data under different shapes.
Each new entry may contain certain metadata tags or readings, and others not.
QuestDB can support on-the-fly data structure changes with minimal overhead.
Whilst the example just above highlights structured data, it is possible for
InfluxDB line protocol users to send data as follows:

```shell
readings,city=London temperature=23.2 1465839830100400000
readings,city=London temperature=23.6 1465839830100700000
readings,make=Honeywell temperature=23.2,humidity=0.443 1465839830100800000
```

This would result in the following table:

| timestamp           | city   | temperature | humidity | make      |
| ------------------- | ------ | ----------- | -------- | --------- |
| 1465839830100400000 | London | 23.5        | NULL     | NULL      |
| 1465839830100700000 | London | 23.6        | NULL     | NULL      |
| 1465839830100800000 | NULL   | 23.2        | 0.358    | Honeywell |

:::tip

Whilst we offer this function for flexibility, we recommend that users try to
minimise structural changes to maintain operational simplicity.

:::

## TCP receiver

The TCP receiver can handle both single and multi-row write requests. It is
fully multi-threaded and customizable. It can work from the common worker pool
or out of dedicated threads. A load balancing mechanism dynamically assigns work
between the threads.

### Overview

By default, QuestDB listens to line protocol packets over TCP on `0.0.0.0:9009`.
If you are running QuestDB with Docker, you will need to publish the port `9009`
using `-p 9009:9009`. This port can be customized.

### Authentication

Although the original protocol does not support it, we have added authentication
over TCP. This works by using an
[elliptic curve P-256](https://en.wikipedia.org/wiki/Elliptic-curve_cryptography)
JSON Web Token (JWT) to sign a server challenge. Details of authentication over
ILP can be found in the
[authentication documentation](/docs/develop/authenticate/)

### Load balancing

A load balancing job reassigns work between threads in order to relieve the
busiest threads and maintain high ingestion speed. It can be triggered in two
ways.

- After a certain number of updates per table
- After a certain amount of time has passed

Once either is met, QuestDB will calculate a load ratio as the number of writes
by the busiest thread divided by the number of writes in the least busy thread.
If this ratio is above the threshold, the table with the least writes in the
busiest worker thread will be reassigned to the least busy worker thread.

![InfluxDB line protocol load balancing diagram](/img/docs/diagrams/influxLineProtocolTCPLoadBalancing.svg)

### Commit strategy

Uncommitted rows are committed either:

- after `line.tcp.maintenance.job.hysterisis.in.ms` milliseconds have passed
- once reaching `line.tcp.max.uncommitted.rows` uncommitted rows.

### Configuration

The TCP receiver configuration can be completely customized using
[configuration keys](/docs/reference/configuration/#influxdb-line-protocol-tcp).
You can use this to configure the tread pool, buffer and queue sizes, receiver
IP address and port, load balancing etc.

### Examples

Please check or Develop section:

- [Authenticate](/docs/develop/authenticate/#influxdb-line-protocol)
- [Insert data](/docs/develop/insert-data/#influxdb-line-protocol)

## UDP receiver

The UDP receiver can handle both single and multi row write requests. It is
currently single-threaded, and performs both network IO and write jobs out of
one thread. The UDP worker thread can work either on its own thread or use the
common thread pool. It supports both multicast and unicast.

### Overview

By default, QuestDB listens for `multicast` line protocol packets over UDP on
`232.1.2.3:9009`. If you are running QuestDB with Docker, you will need to
publish the port `9009` using `-p 9009:9009` and publish multicast packets with
TTL of at least 2. This port can be customized, and you can also configure
QuestDB to listen for `unicast`.

### Commit strategy

Uncommitted rows are committed either:

- after receiving a number of continuous messages equal to
  `line.udp.commit.rate`
- when messages are no longer being received

### Configuration

The UDP receiver configuration can be completely customized using
[configuration keys](/docs/reference/configuration/#influxdb-line-protocol-udp).
You can use this to configure the IP address and port the receiver binds to,
commit rates, buffer size, whether it should run on a separate thread etc.

### Examples

Find an example of how to use this in the
[InfluxDB sender library section](/docs/reference/api/java-embedded/#influxdb-sender-library).
