# Scenario 2: Developing and Debugging a Node.js API by Sharing Code with the Host (Bind Mount)

**Storage type:** Bind Mount
**Use case:** Advanced development workflow with live file sharing

## Overview

We are developing a Node.js project and want code changes made on the host to be reflected inside the container immediately.

## Step 1: Create the project on the host

```bash
mkdir ~/node-app
cd ~/node-app
echo "console.log('API is running');" > app.js
```

## Step 2: Run the Node.js container

```bash
docker run -it \
  --name node-dev \
  -v ~/node-app:/usr/src/app \
  -w /usr/src/app \
  node:16 \
  node app.js
```

## Advanced tip: Read-only bind mount

Append `:ro` to the mount so the container can read the files but cannot modify them:

```bash
docker run -it \
  -v ~/node-app:/usr/src/app:ro \
  -w /usr/src/app \
  node:16 \
  node app.js
```

## Hot reload integration

To apply file changes live inside the container, use a tool such as `nodemon`:

```bash
docker run -it \
  --name node-dev \
  -v ~/node-app:/usr/src/app \
  -w /usr/src/app \
  node:16 \
  bash -c "npm install -g nodemon && nodemon app.js"
```

## Key Takeaways

- A bind mount maps a specific host path directly into the container.
- Edits on the host appear instantly inside the container.
- Use `:ro` when the container should never modify the host files.
- Combine with `nodemon` for an automatic restart on every change.
