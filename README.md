# Data Platform Deployment with Docker Swarm

This project sets up a basic data platform using Docker Swarm across two Fedora Server 41 virtual machines. The platform includes:

- **MinIO** (S3-compatible object storage)
- **Project Nessie** (versioned metadata/catalog service)
- **Trino** (SQL query engine)

## Deployment Overview

The deployment will be tested on the following setup:

### Nodes

- **fedora-node-1** (Swarm manager)
- **fedora-node-2** (Swarm worker)

### VM Specifications

Each Fedora server has the following specifications:

- **Memory**: 2048 MB
- **Storage**: 10 GB
- **Processors**: 4
- **Network**: NAT
- **OS**: Fedora Server 41

### Components

| Component | Description |
|----------|-------------|
| MinIO    | Object storage, S3 compatible |
| Nessie   | Catalog for versioned tables |
| Trino    | Distributed SQL engine |

## Goals

- Deploy each service in Docker Swarm mode
- Validate networking between services across nodes
- Demonstrate querying data in MinIO via Trino using Nessie as the catalog

## Prerequisites

- Proper NAT-based network communication between nodes

## Deployment Steps

_Coming soon..._
