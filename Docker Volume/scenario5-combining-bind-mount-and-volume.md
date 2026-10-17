# Scenario 5: Temporary Data on the Host and Persistent Data in a Volume (Bind Mount + Volume)

**Storage type:** Bind Mount + Volume
**Use case:** Separating temporary working data from final persistent results

## Overview

Suppose we have an application that stores processing data temporarily on the host and moves the final data into a Volume.

## Step 1: Create a volume for the final data

```bash
docker volume create final_data
```

## Step 2: Run the container with both mounts

```bash
docker run -it \
  -v ~/temp_data:/tmp \
  -v final_data:/data \
  data-processor \
  bash
```

## Step 3: Script inside the container

The data is processed and the result is moved into the Volume:

```bash
for file in /tmp/*.csv; do
    cat $file >> /data/final_result.csv
done
```

## Why combine them?

- Temporary data is managed quickly and is directly accessible on the host.
- Final data is stored persistently in the Volume.

## Key Takeaways

- Use a bind mount when you need direct host access to files.
- Use a volume for data that must be managed by Docker and persist reliably.
- Mixing both lets each storage type do what it does best.
