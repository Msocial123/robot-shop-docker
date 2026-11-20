# Running Robot Shop with Podman on Ubuntu

Tested on Ubuntu 26.04 with Podman 5.7 and podman-compose 1.5. It worked both as a normal user
(rootless) and as root, with ufw active.

## 1. Install

```bash
sudo apt-get update
sudo apt-get install -y podman podman-compose git
```

## 2. Get the code

```bash
git clone https://github.com/Msocial123/robot-shop-docker.git
cd robot-shop-docker
```

If you already have a clone, update it first (`git pull`). Older copies still use `debian:10` and
`golang:1.17`, and those no longer build.

## 3. Build and start (recommended: rootless, as your normal user, no sudo)

```bash
podman-compose --podman-build-args=--network=host up -d --build
podman ps
```

Open `http://<server-ip>:8085/`. To use a different port, change `WEB_PORT` in `.env`.

### Running as root (`sudo podman compose ...`)

Use the same command with `sudo`. The `--podman-build-args=--network=host` part is required as root:

```bash
sudo podman-compose --podman-build-args=--network=host up -d --build
```

## 4. Expose the port

Rootless (recommended):

```bash
sudo ufw allow 8085/tcp
```

Root: published ports go through the FORWARD chain, so `ufw allow 8085` is **not** enough:

```bash
sudo ufw route allow proto tcp to 10.253.7.0/24 port 8080
```

On a cloud VM, also add an inbound TCP 8085 rule to the security group / firewall.

## Why the build fails with "Temporary failure in name resolution"

Symptoms: `npm ERR! EAI_AGAIN`, `Temporary failure resolving 'deb.debian.org'`,
`apk ... DNS: transient error`, `read udp 10.88.0.x -> <dns>:53: i/o timeout`.

As root, build containers run on the Podman bridge (`podman0`, 10.88.0.0/16). Their DNS and
download traffic is *routed* through the host. Ubuntu's ufw default is `deny (routed)`, so that
traffic is dropped (`[UFW BLOCK] IN=podman0 OUT=<nic> ... DPT=53` in `journalctl -k`).

Fixes, any one of:

* Build with the host network: `--podman-build-args=--network=host` (as above).
  Docker Compose uses `build.network: host`, which is already set in `docker-compose.yaml`.
  podman-compose ignores that key, so the flag is needed.
* Run rootless. Rootless networking (pasta / slirp4netns) does not go through ufw's FORWARD chain.
* Allow routed traffic from Podman bridges: `sudo ufw route allow in on podman0`
  (plus `sudo ufw allow in on podman+ to any port 53` for container-to-container DNS).

At runtime the stack does not depend on container DNS. Each service has a fixed IP
(`RS_SUBNET_PREFIX` in `.env`, default `10.253.7`) and an `/etc/hosts` entry for every other
service, so `mongodb`, `mysql`, `rabbitmq` and the other service names resolve even when ufw
blocks DNS.

## Other portability changes

* All images are fully qualified (`docker.io/library/...`). A fresh Ubuntu Podman has no
  unqualified-search registry, so short names such as `redis:6.2` fail with
  "short-name did not resolve".
* The database uses `mariadb:10.11` instead of `mysql:5.7`. On hosts that have MySQL installed,
  the host AppArmor profile for `/usr/sbin/mysqld` also confines the container's mysqld, and
  `podman stop` hangs. `mysql:5.7` is also amd64-only.

## Useful commands

```bash
podman-compose ps
podman-compose logs -f web
podman-compose down             # stop and remove containers
loginctl enable-linger $USER    # keep rootless containers running after you log out
```
