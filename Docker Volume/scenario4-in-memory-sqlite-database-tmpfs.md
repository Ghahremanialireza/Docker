# Scenario 4: In-Memory Database for Fast Testing (tmpfs)

**Storage type:** tmpfs
**Use case:** Running a throwaway SQLite database in RAM for quick tests

## Overview

Suppose we want to run a SQLite database for testing, with the data stored temporarily in RAM.

## Step 1: Run the SQLite container with tmpfs

```bash
docker run -it \
  --tmpfs /data \
  -e SQLITE_FILE=/data/test.db \
  nouchka/sqlite3 \
  bash
```

## Step 2: Run queries in RAM

Inside the container:

```bash
sqlite3 /data/test.db "CREATE TABLE test (id INT, name TEXT); INSERT INTO test VALUES (1, 'Docker');"
sqlite3 /data/test.db "SELECT * FROM test;"
```

## Advantages

- High speed because RAM is used.
- Data is erased when the container ends.

## Key Takeaways

- Mounting `tmpfs` at `/data` makes the database file live entirely in memory.
- This is perfect for disposable test databases where persistence is not needed.
