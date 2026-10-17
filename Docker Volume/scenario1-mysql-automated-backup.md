# Scenario 1: Running MySQL with Automated Backups (Volume)

**Storage type:** Docker Volume
**Use case:** Persistent data management in a production environment

## Overview

Suppose we need a MySQL container whose data is backed up periodically. We use two named volumes: one for the live database files and one for the backup archives.

## Step 1: Create the volumes

```bash
docker volume create mysql_data
docker volume create mysql_backup
```

## Step 2: Run the MySQL container with the volume

```bash
docker run -d \
  --name mysql-container \
  -v mysql_data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=my-secret \
  mysql:8.0
```

## Step 3: Run the backup container

Every night at 2 AM, we run a throwaway container that archives the database data into the backup volume:

```bash
docker run --rm \
  -v mysql_data:/source \
  -v mysql_backup:/backup \
  ubuntu \
  tar cvf /backup/mysql_backup_$(date +%Y%m%d).tar /source
```

## Step 4: Restore the data

If a backup needs to be restored:

```bash
docker run --rm \
  -v mysql_data:/source \
  -v mysql_backup:/backup \
  ubuntu \
  tar xvf /backup/mysql_backup_20231223.tar -C /source
```

## Key Takeaways

- Named volumes are managed by Docker and survive container removal.
- A `--rm` helper container is a clean way to run one-off backup/restore jobs.
- Mounting both volumes into the same helper container lets you copy data between them without touching the host filesystem.
