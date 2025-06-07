# Data Platform Deployment with Docker Swarm

This project sets up a basic data platform using Docker Swarm across two Fedora Server 42 virtual machines. The platform includes:

- **MinIO**: S3-compatible object storage
- **Project Nessie**: Versioned metadata/catalog service
- **Trino**: Distributed SQL query engine

## Deployment Overview

### Nodes

- **fedora-node-1** – Docker Swarm Manager
- **fedora-node-2** – Docker Swarm Worker

### VM Specifications

- **Memory**: 2048 MB  
- **Storage**: 10 GB  
- **Processors**: 4  
- **Network**: NAT Network  
- **Operating System**: Fedora Server 42

### Components

| Component | Description                        |
|-----------|------------------------------------|
| MinIO     | Object storage (S3-compatible)     |
| Nessie    | Catalog for versioned tables       |
| Trino     | Distributed SQL engine             |

## Goals

- Deploy each service in Docker Swarm mode
- Ensure inter-node networking is functional
- Query data stored in MinIO using Trino and Nessie as the catalog

## Prerequisites

- NAT network connectivity between nodes must be verified

## IP Address

| Node Name     | IP Address  |
|---------------|-------------|
| fedora-node-1 | 10.0.2.15   |
| fedora-node-2 | 10.0.2.4    |

## Port Forwarding

| Description | Host Port | Node Name     | Node Port |
|-------------|-----------|---------------|-----------|
| SSH         | 5000      | fedora-node-1 | 22        |
| SSH         | 6000      | fedora-node-2 | 22        |

---

## Deployment Steps

### Step 1: SSH into Nodes

Enable SSH access by setting up port forwarding from the host machine to each VM:

- Host port `5000` → `fedora-node-1` SSH port (22)
- Host port `6000` → `fedora-node-2` SSH port (22)

### Step 2: Install Docker

Install Docker on **each** Fedora VM:

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf-3 config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Enable Docker to start automatically:

```bash
sudo systemctl enable --now docker
```

### Step 3: Initialize Docker Swarm Manager

Run the following command on `fedora-node-1` to initialize the Docker Swarm manager:

```bash
sudo docker swarm init --advertise-addr 10.0.2.15
```

Sample output:
```text
Swarm initialized: current node (...) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token <SWARM_TOKEN> 10.0.2.15:2377
```

If you need to retrieve the token later:

```bash
sudo docker swarm join-token worker
```

### Step 4: Join Docker Swarm Worker

Run this on `fedora-node-2`:

```bash
sudo docker swarm join --token <SWARM_TOKEN> 10.0.2.15:2377
```

If you get the error:

```text
Error response from daemon: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp 10.0.2.15:2377: connect: no route to host
```

Ping the manager node:

```bash
ping 10.0.2.15
```

If ping succeeds, the issue is likely the firewall. On `fedora-node-1`, allow the required port:

```bash
sudo firewall-cmd --permanent --add-port=2377/tcp
sudo firewall-cmd --permanent --add-port=7946/tcp
sudo firewall-cmd --permanent --add-port=7946/udp
sudo firewall-cmd --reload
```

Then retry joining the swarm on the worker node.

Sample output:

```bash
sudo docker swarm join --token SWMTKN-1-369wciwq2vp48e1fn46b6pnufypzikr6e1e0zqwxlamlvw5c4n-cozbhcgcy02hwxaap80xq2kjr 10.0.2.15:2377
This node joined a swarm as a worker.
```


### Step 5: Verify Swarm Status

Run the following on the manager node:

```bash
sudo docker node ls
```

Sample output:

```text
ID                            HOSTNAME                STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
cxv90mxb0zypcedv18ng8vgwz *   localhost.localdomain   Ready     Active         Leader           28.2.2
g7z9ya6bo5268zhlto2o3exoc     localhost.localdomain   Ready     Active                          28.2.2
```

### Step 7: Test Docker Swarm with a Simple Service

To confirm that Docker Swarm is working correctly across both nodes, I will deploy a simple test service.

We'll use the `alpine` image to create a service that pings `google.com` from each replica:

```bash
sudo docker service create --replicas 2 --name helloworld alpine ping google.com
```

The command above

- Deploys a service called helloworld

- Runs 2 replicas of the service (which should be distributed across the nodes)

- Uses the alpine image

- Executes ping google.com inside each container


and checking it status 

```
sudo docker service ls
```

Sample output:

```
ID             NAME         MODE         REPLICAS   IMAGE           PORTS
3zgq7abtjfxk   helloworld   replicated   2/2        alpine:latest
```