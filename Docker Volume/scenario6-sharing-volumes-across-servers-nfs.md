# Scenario 6: Sharing a Volume Between Multiple Servers with NFS (Advanced Volume Management)

**Storage type:** Volume with the NFS driver option
**Use case:** Shared storage across multiple Docker hosts

## Overview

Suppose we have several Docker servers and want the data of one Volume to be shared among all of them.

## Step 1: Set up NFS on the main server

On the NFS server:

```bash
apt install nfs-kernel-server
mkdir /docker-volumes
echo "/docker-volumes *(rw,sync,no_subtree_check)" >> /etc/exports
exportfs -a
```

## Step 2: Mount the volume in Docker

On each Docker server:

```bash
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=<nfs-server-ip>,rw \
  --opt device=:/docker-volumes/my-volume \
  shared_volume
```

## Step 3: Share the volume between multiple containers

```bash
docker run -v shared_volume:/data --name app1 myapp
docker run -v shared_volume:/data --name app2 myapp
```

## Key Takeaways

- The `local` driver can mount remote NFS shares via `--opt type=nfs`.
- Every Docker host that creates the same volume definition sees the same data.
- Multiple containers can mount the same volume at once.
- In production, restrict the `/etc/exports` entry to trusted IPs instead of `*`.
