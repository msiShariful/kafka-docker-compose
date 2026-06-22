# Apache Kafka on Docker Compose (KRaft)

> A production-style Apache Kafka cluster on Docker Compose using **KRaft** — no ZooKeeper. Three controllers, three brokers, and a Kafka-UI dashboard, all fully configurable via environment variables.

![Apache Kafka](https://img.shields.io/badge/Apache%20Kafka-231F20?logo=apachekafka&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![Docker Compose](https://img.shields.io/badge/Docker%20Compose-2496ED?logo=docker&logoColor=white)
![KRaft](https://img.shields.io/badge/Mode-KRaft%20(no%20ZooKeeper)-blue)
![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)

## Overview

This repository spins up a highly available Apache Kafka cluster running in **KRaft mode** (Kafka Raft metadata mode), which removes the dependency on ZooKeeper entirely. The cluster is composed of:

- **3 dedicated controllers** that form the KRaft metadata quorum.
- **3 dedicated brokers** that handle client traffic, configured for high availability.
- **Kafka-UI**, a web dashboard for inspecting topics, consumer groups, and messages.

Separating the controller and broker roles mirrors a production-style topology and lets you reason about quorum and data planes independently.

## Features

- **KRaft quorum** — three controllers form the Raft metadata quorum; no ZooKeeper required.
- **High availability** — default replication factor of `3`, minimum in-sync replicas of `2`, and replicated offset/transaction topics so the cluster tolerates a broker outage without data loss.
- **Kafka-UI dashboard** — browse topics, partitions, consumer groups, and produce/consume messages from the browser.
- **Fully env-configurable** — image tags, heap sizes, cluster ID, and the external host are all driven by `.env`.

## Architecture

The stack defines seven services:

| Service | Role | Node ID | Image | Host Port(s) |
| ----------- | ------------------ | ------- | ------------------------------ | ------------ |
| controller-1 | KRaft controller | 1 | `apache/kafka-native` | — (9093 internal) |
| controller-2 | KRaft controller | 2 | `apache/kafka-native` | — (9093 internal) |
| controller-3 | KRaft controller | 3 | `apache/kafka-native` | — (9093 internal) |
| broker-1 | Broker | 4 | `apache/kafka-native` | 29092 |
| broker-2 | Broker | 5 | `apache/kafka-native` | 39092 |
| broker-3 | Broker | 6 | `apache/kafka-native` | 49092 |
| kafka-ui | Web dashboard | — | `provectuslabs/kafka-ui` | 9021 |

- **Controllers** run with `KAFKA_PROCESS_ROLES=controller`, listen on port `9093`, and form the quorum `1@controller-1:9093,2@controller-2:9093,3@controller-3:9093`. Default heap: `-Xms1G -Xmx1G`.
- **Brokers** run with `KAFKA_PROCESS_ROLES=broker`, expose an internal listener `PLAINTEXT://:19092` (used cluster-internally) and a host listener `PLAINTEXT_HOST://:9092` mapped to the host ports above. They depend on the three controllers. Default heap: `-Xms2G -Xmx2G`.
- **Kafka-UI** connects to the cluster named `local` via the internal bootstrap servers `broker-1:19092,broker-2:19092,broker-3:19092`.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) (Engine 20.10+)
- [Docker Compose](https://docs.docker.com/compose/) v2 (`docker compose`)
- **Sufficient RAM.** Each broker defaults to a `2G` heap and each controller to `1G`, so the full stack requests roughly `9G` of JVM heap before OS/container overhead. Make sure Docker is allocated enough memory (12&nbsp;GB+ recommended), or lower the heap values in `.env`.

## Quick Start

1. Start the cluster:

   ```bash
   docker compose up -d
   ```

2. Wait for the controllers and brokers to become healthy, then open Kafka-UI:

   ```
   http://localhost:9021
   ```

3. Select the `local` cluster to explore topics, partitions, and consumer groups.

## Connecting

Clients on your host machine connect through the per-broker external (host) listeners:

| Broker | External bootstrap | Internal bootstrap |
| -------- | ------------------ | ------------------ |
| broker-1 | `localhost:29092`  | `broker-1:19092`   |
| broker-2 | `localhost:39092`  | `broker-2:19092`   |
| broker-3 | `localhost:49092`  | `broker-3:19092`   |

- From your **host**, use any of `localhost:29092`, `localhost:39092`, `localhost:49092` as the bootstrap server.
- From **inside the Docker network** (other containers, or `docker exec` into a broker), use the internal listeners `broker-N:19092`.

## Configuration

All settings live in `.env`:

| Variable | Default | Description |
| ------------------------------- | ------------------- | ----------- |
| `KAFKA_IMAGE_TAG` | `latest` | Tag for the `apache/kafka-native` image used by controllers and brokers. |
| `KAFKA_UI_TAG` | `latest` | Tag for the `provectuslabs/kafka-ui` image. |
| `CLUSTER_ID` | `5L6g3nShT-eMCtK--X86sw` | KRaft cluster ID. Generate a fresh one for any non-throwaway cluster. |
| `KAFKA_EXTERNAL_HOST` | `localhost` | Hostname advertised to external clients. |
| `KAFKA_HEAP_OPTS` | `-Xms1G -Xmx1G` | JVM heap options for the controllers. |
| `KAFKA_BROKER_HEAP_OPTS` | `-Xms2G -Xmx2G` | JVM heap options for the brokers. |
| `KAFKA_AUTO_CREATE_TOPICS_ENABLE` | `false` | Whether brokers auto-create topics on first reference. |

Replication and durability defaults baked into the brokers: `offsets.topic.replication.factor=3`, `transaction.state.log.replication.factor=3`, `min.insync.replicas=2`, `default.replication.factor=3`.

## Create a Topic / Produce / Consume

Exec into any broker container and use the bundled CLI tools (pointing at the broker's internal listener on `19092`).

Create a topic:

```bash
docker exec -it broker-1 /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:19092 \
  --create --topic demo --partitions 3 --replication-factor 3
```

List topics:

```bash
docker exec -it broker-1 /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:19092 --list
```

Produce messages (type lines, then Ctrl+C to exit):

```bash
docker exec -it broker-1 /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:19092 --topic demo
```

Consume messages from the beginning:

```bash
docker exec -it broker-2 /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:19092 --topic demo --from-beginning
```

## Tear Down

Stop and remove the containers:

```bash
docker compose down
```

Remove containers **and** wipe all Kafka data volumes:

```bash
docker compose down -v
```

## Security

All listeners in this stack use `PLAINTEXT` with **no authentication, authorization, or TLS**, so it is intended for local and development use only. Do not expose the broker host ports (`29092`, `39092`, `49092`) or the Kafka-UI port (`9021`) to untrusted networks. See [SECURITY.md](SECURITY.md) for the full security policy and hardening guidance for networked deployments.

## Contributing

Contributions are welcome! Please open an issue to discuss a change, or submit a pull request against the `master` branch. Keep `docker-compose.yml` and the `.env` defaults in sync with this README.

## License

This project is licensed under the [MIT License](LICENSE).
