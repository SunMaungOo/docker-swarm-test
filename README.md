# Data Platform Deployment with Docker Swarm

This project sets up a basic data platform using Docker Swarm across two Fedora Server 42 virtual machines. The platform includes:

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
- **Network**: NAT Network
- **OS**: Fedora Server 42

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

## IP Address

|Node Name | IP Address   |
|-------------|-----------|
| fedora-node-1|  10.0.2.15 |
| fedora-node-2 | 10.0.2.4  |

## Port Forwarding

| Description | Host Port | Node Name | Node Port   |
|-------------|-----------|-----------|-------------|
| SSH         | 5000      | fedora-node-1 | 22
| SSH         | 6000      | fedora-node-2 | 22


## Deployment Steps

### SSH 

The deployment will be done by SSH so I first needed to start up the port forward to the virtual machine from our host machine. 
I redirect the port ``5000`` of host machine to ``fedora-node-1`` ssh port and port ``6000`` of host machine to ``fedora-node-2`` ssh port

### Docker 

We first needed to install docker in each virtual machine by 

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf-3 config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Additionally I wanted the docker to start up when our virtual machine boot up

```bash 
sudo systemctl enable --now docker
```


