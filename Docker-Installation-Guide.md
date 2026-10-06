# Docker Installation Guide

A clean, up-to-date walkthrough for installing Docker Engine and Docker Desktop, based on the official Docker documentation — followed by a practical section on working around registry access restrictions, which commonly affect users in sanctioned regions such as Iran.

Replace anything in `<angle_brackets>` with your own value.

## Contents

- [Before you start](#before-you-start)
- [Ubuntu and Debian](#ubuntu-and-debian)
- [Fedora, CentOS, and RHEL](#fedora-centos-and-rhel)
- [macOS](#macos)
- [Windows](#windows)
- [Post-install configuration](#post-install-configuration)
- [Verifying the installation](#verifying-the-installation)
- [Updating Docker](#updating-docker)
- [Removing Docker](#removing-docker)
- [Working around registry access restrictions](#working-around-registry-access-restrictions)
- [Troubleshooting](#troubleshooting)

---

## Before you start

Docker Engine runs on 64-bit systems (x86_64 or arm64) with a kernel of 3.10 or newer, which rules out essentially nothing released in the last decade. You'll need administrative access on the machine — `sudo` on Linux, an admin account on macOS or Windows.

Officially supported Linux distributions change over time, so if you're on something unusual it's worth checking the current list rather than trusting a guide written months ago:

https://docs.docker.com/engine/install/

One thing worth knowing upfront: installing Docker and being able to pull images from Docker Hub are two separate problems. The install can succeed cleanly and you can still hit a wall the first time you run `docker pull`. Keep that distinction in mind if you run into trouble later in this guide.

---

## Ubuntu and Debian

### Remove conflicting packages

If you've installed Docker before, or your distribution shipped something Docker-adjacent out of the box, clear it out first:

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt-get remove $pkg
done
```

It's fine if some of these were never installed — `apt-get` will just say so and move on.

### Set up the repository

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
```

On Debian, swap `ubuntu` for `debian` in both URLs above.

### Install

```bash
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

This pulls in the engine itself (`docker-ce`), the CLI, the `containerd` runtime, and the Buildx and Compose plugins — everything you'd actually use day to day.

Avoid `apt install docker.io`. It's the version Ubuntu packages on its own schedule, and it tends to lag well behind what Docker actually ships, security patches included.

If you just need something running quickly for a one-off test, Docker also publishes a convenience script that detects your distro and handles the above automatically:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

It's not what you want for a server you intend to keep around, but it's fine for throwaway environments.

---

## Fedora, CentOS, and RHEL

```bash
sudo dnf remove docker docker-client docker-client-latest docker-common \
  docker-latest docker-latest-logrotate docker-logrotate docker-engine

sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/fedora/docker-ce.repo

sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl enable --now docker
```

On CentOS or RHEL, use `centos` instead of `fedora` in the repo URL.

---

## macOS

Download Docker Desktop for your chip (Apple Silicon or Intel) from:

https://www.docker.com/products/docker-desktop/

Open the `.dmg`, drag Docker into Applications, then launch it from Launchpad. It'll ask for a few system permissions on first run — networking, a privileged helper — approve those, and once the whale icon in the menu bar stops animating, you're ready:

```bash
docker run hello-world
```

If you'd rather manage it through Homebrew:

```bash
brew install --cask docker
```

You still need to open Docker.app once after installing — Homebrew puts it in place, but it won't start the engine for you.

---

## Windows

WSL 2 is a prerequisite:

```powershell
wsl --install
```

Reboot after that finishes.

Then grab the installer from the same page as above, run it, and leave **Use WSL 2 instead of Hyper-V** checked — it's the default and the right choice for almost everyone. Reboot once more after installation, open Docker Desktop, wait for the engine to come up, and confirm from PowerShell:

```powershell
docker run hello-world
```

---

## Post-install configuration

### Running Docker without sudo (Linux)

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```

Log out and back in if `newgrp` doesn't seem to take effect immediately.

### Starting Docker on boot

```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

---

## Verifying the installation

```bash
docker --version
docker compose version
docker buildx version
docker run hello-world
systemctl is-active docker
docker info
```

A successful `hello-world` run means everything between the engine, the CLI, and the registry connection is working end to end.

---

## Updating Docker

**Ubuntu/Debian:**
```bash
sudo apt-get update
sudo apt-get install --only-upgrade docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**Fedora/CentOS:**
```bash
sudo dnf update docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**macOS/Windows:** Docker Desktop checks for updates on its own and will prompt you, or you can trigger a check manually from its settings menu.

---

## Removing Docker

```bash
sudo apt-get purge -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras

sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
sudo rm -rf /etc/docker
sudo rm /etc/apt/sources.list.d/docker.list
sudo rm /etc/apt/keyrings/docker.asc
```

The two `rm -rf` lines above delete every image, container, and volume you've ever created. There's no undo — make sure that's actually what you want before running them.

---

## Working around registry access restrictions

Users connecting from Iran and a handful of other sanctioned regions typically run into two distinct failures, not one, and it helps to treat them separately:

- `download.docker.com` is unreachable or painfully slow, which blocks the **install** steps above.
- `registry-1.docker.io` is unreachable, which blocks `docker pull` **after** installation succeeds.

If you installed Docker without trouble but `docker pull hello-world` hangs or times out, you're dealing with the second problem specifically — skip straight to the registry mirror section below rather than troubleshooting the install itself.

### Option 1 — Registry mirror (the most stable fix)

This tells the Docker daemon to route pulls through an alternate endpoint instead of talking to Docker Hub directly. It doesn't require a VPN to stay connected in the background, which makes it the option worth trying first.

Edit (or create) the daemon configuration file:

```bash
sudo mkdir -p /etc/docker
sudo nano /etc/docker/daemon.json
```

```json
{
  "registry-mirrors": [
    "https://docker.arvancloud.ir",
    "https://mirror.gcr.io"
  ]
}
```

Mirror addresses run by Iranian providers change fairly often — some go dark, others get introduced. Treat the two above as a starting point, not a permanent answer, and verify they're actually reachable before relying on them:

```bash
curl -v https://docker.arvancloud.ir/v2/
```

Apply the change:

```bash
sudo systemctl daemon-reexec
sudo systemctl restart docker
```

Confirm the mirror is in effect, then test it:

```bash
docker info | grep -A 3 "Registry Mirrors"
docker pull alpine
```

### Option 2 — Proxy the Docker daemon

If you already run a proxy or VPN client (V2Ray, WARP, an SSH tunnel), note that Docker's daemon does **not** inherit your system or browser proxy settings automatically — it has to be configured separately, at the systemd service level.

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo nano /etc/systemd/system/docker.service.d/http-proxy.conf
```

```ini
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:<proxy_port>"
Environment="HTTPS_PROXY=http://127.0.0.1:<proxy_port>"
Environment="NO_PROXY=localhost,127.0.0.1"
```

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl show --property=Environment docker
```

For a quick one-off pull without touching the service at all, exporting the variables in your shell is enough:

```bash
export HTTP_PROXY=http://127.0.0.1:<proxy_port>
export HTTPS_PROXY=http://127.0.0.1:<proxy_port>
docker pull <image_name>
```

On macOS or Windows, the equivalent setting lives in Docker Desktop under **Settings → Resources → Proxies**. Enter the proxy's host and port there and apply.

### Option 3 — Proxy the package manager during install

If `download.docker.com` itself is unreachable — meaning the install steps are what's failing, not the pull — a temporary proxy on `apt` gets you through it:

```bash
sudo nano /etc/apt/apt.conf.d/proxy.conf
```

```text
Acquire::http::Proxy "http://127.0.0.1:<proxy_port>";
Acquire::https::Proxy "http://127.0.0.1:<proxy_port>";
```

Remove the file once installation is done if you don't want `apt` routing through the proxy permanently:

```bash
sudo rm /etc/apt/apt.conf.d/proxy.conf
```

Some Iranian cloud providers (ArvanCloud and Parspack among them) also run mirrored `apt`/`dnf` repositories that include Docker's packages. If your server already sits in a domestic data center, pointing at one of these is usually faster and more reliable than tunneling through a proxy.

### Option 4 — Pull on another machine, transfer manually

When neither a mirror nor a proxy is working reliably, the fallback is to pull the image somewhere that has unrestricted access, export it, and carry it over:

```bash
# On the machine with access
docker pull <image_name>:<tag>
docker save <image_name>:<tag> | gzip > <file_name>.tar.gz
```

Transfer the file however is convenient — `scp`, a shared drive, physical media — then load it locally:

```bash
gunzip -c <file_name>.tar.gz | docker load
docker images
```

It's manual and doesn't scale to a workflow where you're pulling new images regularly, but it's reliable when nothing else is.

---

## Troubleshooting

| Symptom | Likely cause and fix |
|---|---|
| `Get "https://registry-1.docker.io/...": context deadline exceeded` | Docker Hub itself is unreachable — try the registry mirror or proxy options above |
| `Could not resolve host: download.docker.com` | DNS resolution is being blocked, not just the connection — try a different resolver (`1.1.1.1`, or a domestic alternative) or route through a proxy |
| Edited `daemon.json` but nothing changed | Changes don't take effect until the service restarts: `sudo systemctl restart docker` |
| Proxy works from the shell but `docker pull` still fails | Shell environment variables don't reach the daemon — the proxy needs to be set at the systemd service level, not just exported in your terminal |
| A mirror that used to work now times out | Community and provider-run mirrors go offline without warning fairly often — test a couple of alternatives rather than assuming one is permanently broken |
| Certificate errors against a mirror or proxy | Confirm the mirror URL is actually `https` and reachable: `curl -v https://<mirror_url>/v2/` |
| Install completed but `hello-world` won't pull | You're not dealing with an install problem — go straight to the registry access section above |

---

## References

- [Install Docker Engine](https://docs.docker.com/engine/install/)
- [Install on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- [Install on Debian](https://docs.docker.com/engine/install/debian/)
- [Install on Fedora](https://docs.docker.com/engine/install/fedora/)
- [Docker Desktop for Mac](https://docs.docker.com/desktop/install/mac-install/)
- [Docker Desktop for Windows](https://docs.docker.com/desktop/install/windows-install/)
- [Configuring the daemon to use a proxy](https://docs.docker.com/config/daemon/systemd/#httphttps-proxy)
- [Registry mirror configuration](https://docs.docker.com/docker-hub/image-library/mirror/)
