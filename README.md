# openshift-demo

## Overview

This repository illustrates reference implementations of Senzing on OpenShift.

The instructions show how to set up a system that:

1. Reads JSON lines from a file on the internet.
1. Sends each JSON line as a message to a queue.
1. Reads messages from the queue and inserts into Senzing.
1. Reads information from Senzing via [Senzing API Server](https://github.com/Senzing/senzing-api-server) server.
1. Views resolved entities in a [web app](https://github.com/Senzing/entity-search-web-app).

The following diagram shows the relationship of the Helm charts, docker containers, and code in this OpenShift demonstration.

![Image of architecture](docs/img-architecture/architecture.png)

## Implementation

The following table indicates the instructions for variations in components.

1. Component variants:
    1. Queue
        1. RabbitMQ
    1. Database
        1. Db2
1. Implementations of the docker formation:

    | Queue    | Database   | Instructions |
    |----------|------------|:------------:|
    | RabbitMQ | Db2        | [:page_facing_up:](docs/helm-rabbitmq-db2/README.md) |
