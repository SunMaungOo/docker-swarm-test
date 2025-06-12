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

## Step 8: Set Up CephFS for Distributed File System

In a Docker **Swarm** environment, managing data volumes is different from standalone **Docker**. If you define a volume using the `volume` keyword like in Docker, each node will create and manage its own volume independently. This means only the container/task is replicated—not the data.

To solve this, we need a shared volume accessible from all nodes. We’ll use **CephFS**, a POSIX-compliant distributed file system built on Ceph's object store.

### Set Hostnames (if not already set)

Assign hostnames to the nodes:

```bash
sudo hostnamectl set-hostname fedora-node-1  # Manager node
sudo hostnamectl set-hostname fedora-node-2  # Worker node
```

### Overview

CephFS requires a cluster setup. We'll configure:

- `fedora-node-1` as the **manager** node
- `fedora-node-2` as the **worker** node

## Step 9: Set Up Ceph Cluster (Manager Node)

First, install `cephadm` on the manager node:

```bash
curl -sLO https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
chmod +x cephadm
```

Then bootstrap the Ceph cluster:

```bash
MYIP=$(ip route get 1.1.1.1 | grep -oP 'src \K\S+')
sudo mkdir -p /etc/ceph
sudo ./cephadm bootstrap --mon-ip $MYIP
```

Watch the logs during setup. If you see:

```text
podman|docker (/usr/sbin/docker) is present
```

…it means Ceph will run using Docker.

A dashboard will also be deployed automatically:

```text
Ceph Dashboard is now available at:

     URL: https://fedora-node-1:8443/
    User: admin
 Password: pti07yg1m0
```

## Step 10: Set Up Root Certificate Access (Manager & Worker)

To allow Ceph to control the worker node, configure SSH root access from the manager:

### On the worker node:

```bash
sudo ./cephadm install ceph-common
```

### On the manager node:

```bash
sudo ssh-copy-id -f -i /etc/ceph/ceph.pub root@10.0.2.4
```

Switch to the root user to generate and copy SSH keys:

```bash
su
ssh-keygen
ssh-copy-id root@10.0.2.4
exit
```

## Step 11: Add Worker Node to the Cluster (Manager Node)

Now, add the worker node to the Ceph cluster:

```bash
sudo ceph orch host add fedora-node-2 10.0.2.4
```

You should see:

```text
Added host 'fedora-node-2'
```

Verify with:

```bash
sudo ceph orch host ls
```

Expected output:

```text
HOST           ADDR           LABELS  STATUS  
fedora-node-1  fedora-node-1                  
fedora-node-2  10.0.2.4
```

## Step 12: Configure OSDs (Manager Node)

OSDs (Object Storage Daemons) are the actual storage units used by Ceph.

### List available storage devices:

```bash
sudo ceph orch device ls
```

Look for devices marked as `Available: Yes`.

Example:

```text
Hostname       Path        Type  Serial               Size   Health   Available  
fedora-node-1  /dev/sdb    hdd   VB2737e02b-9082c244  7516M  Unknown  Yes        
fedora-node-2  /dev/sdb    hdd   VB48a76328-9a5f4e0d  7516M  Unknown  Yes        
```

### Dry run to preview OSD setup:

```bash
sudo ceph orch apply osd --all-available-devices --dry-run
```

Sample output:

```text
+---------+-----------------------+---------------+----------+----+-----+
|SERVICE  |NAME                   |HOST           |DATA      |DB  |WAL  |
+---------+-----------------------+---------------+----------+----+-----+
|osd      |all-available-devices  |fedora-node-1  |/dev/sdb  |-   |-    |
|osd      |all-available-devices  |fedora-node-2  |/dev/sdb  |-   |-    |
+---------+-----------------------+---------------+----------+----+-----+
```

### Apply the OSD configuration:

```bash
sudo ceph orch apply osd --all-available-devices
```

### Verify OSDs:

```bash
sudo ceph osd tree
```

Example:

```text
ID  CLASS  WEIGHT   TYPE NAME            STATUS  REWEIGHT  PRI-AFF
-1         0.01358  root default                                 
-3         0.00679      host fedora-node-1                       
 0    hdd  0.00679          osd.0          up   1.00000  1.00000
-5         0.00679      host fedora-node-2                       
 1    hdd  0.00679          osd.1          up   1.00000  1.00000
```

If OSD creation fails, you may need to wipe the device:

```bash
sudo wipefs --all /dev/sdb
```

> ⚠️ **Warning:** This will erase **all data** on the device.

## Step 13: Create a Ceph File System (Manager Node)

Create the CephFS file system:

```bash
sudo ceph fs volume create ceph-fs-data
```

Verify it was created:

```bash
sudo ceph fs ls
```

Expected output:

```text
name: ceph-fs-data, metadata pool: cephfs.ceph-fs-data.meta, data pools: [cephfs.ceph-fs-data.data]
```

## Step 14: Prepare the Worker Node (Worker Node)

Install the Ceph client tools:

```bash
sudo dnf install ceph-common
```

## Step 15: Copy Ceph Configuration (Manager Node)

Copy the cluster configuration files to the worker node:

```bash
sudo scp /etc/ceph/* root@10.0.2.4:/etc/ceph
```

## Step 16: Mount the Ceph File System (Both Nodes)

### On both nodes:

1. Create a mount point:

```bash
sudo mkdir /mnt/ceph
```

2. Mount the Ceph file system:

```bash
sudo mount -t ceph 10.0.2.15:/ /mnt/ceph -o name=admin,conf=/etc/ceph/ceph.conf,mds_namespace=ceph-fs-data
```

3. Verify the mount:

```bash
mount | grep ceph
```

Expected output:

```text
10.0.2.15:/ on /mnt/ceph type ceph (rw,...,mds_namespace=ceph-fs-data)
```

✅ **Success!** Files placed in `/mnt/ceph` will now be **synchronously replicated** between both nodes in **both directions**.

