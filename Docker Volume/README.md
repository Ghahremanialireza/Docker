# Docker Storage Scenarios

Practical, hands-on examples for Docker storage: **Volumes**, **Bind Mounts**, and **tmpfs**, including combinations and production-oriented setups.

## Scenarios

| # | Scenario | Storage Type |
|---|----------|--------------|
| 1 | [MySQL with automated backups](scenario1-mysql-automated-backup.md) | Volume |
| 2 | [Node.js development with live code sharing](scenario2-nodejs-development-bind-mount.md) | Bind Mount |
| 3 | [Fast temporary file processing](scenario3-tmpfs-temporary-file-processing.md) | tmpfs |
| 4 | [In-memory SQLite database for testing](scenario4-in-memory-sqlite-database-tmpfs.md) | tmpfs |
| 5 | [Temporary host data + persistent volume](scenario5-combining-bind-mount-and-volume.md) | Bind Mount + Volume |
| 6 | [Sharing a volume across servers with NFS](scenario6-sharing-volumes-across-servers-nfs.md) | Volume (NFS) |

## When to Use What

- **Volume:** persistent data managed by Docker (databases, backups, shared storage).
- **Bind Mount:** direct access to host files (development, hot reload).
- **tmpfs:** fast, temporary data stored in RAM, erased when the container stops.

## Prerequisites

- Docker installed and running
- Basic familiarity with the command line
