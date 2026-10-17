# Scenario 3: Caching Processing Files with High Speed (tmpfs)

**Storage type:** tmpfs (combined with bind mounts for input/output)
**Use case:** Fast temporary storage for intermediate files

## Overview

Suppose we have an image-processing application that needs to read and write temporary files very quickly. We keep the intermediate files in RAM using `tmpfs`, while input and output folders are bind-mounted from the host.

## Step 1: Run the container with tmpfs

```bash
docker run -it \
  --tmpfs /tmp \
  -v ~/images:/input \
  -v ~/processed:/output \
  image-processor \
  bash
```

## Step 2: Processing script

Inside the container, run the following script:

```bash
for img in /input/*.jpg; do
    convert $img -resize 50% /tmp/$(basename $img)
    mv /tmp/$(basename $img) /output
done
```

## Advantage of tmpfs

Because `tmpfs` lives in RAM, reading, writing, and moving temporary files is very fast.

## Key Takeaways

- `tmpfs` data is stored in memory and never written to disk.
- Contents are lost when the container stops.
- It is ideal for scratch space such as temporary processing files.
