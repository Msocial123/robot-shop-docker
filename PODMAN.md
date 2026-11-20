# Running Robot Shop with Podman on Ubuntu

The Compose file is generic. Anything specific to one machine (port, firewall) is set in `.env` or on the host.

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

## 3. Build and start

As your normal user (rootless, recommended) or with `sudo`, the same command works:

```bash
podman-compose up -d --build        # or: podman compose up -d --build / docker compose up -d --build
podman ps
```

Open `http://<server-ip>:8085/`. To use another port, change `WEB_PORT` in `.env`.

## 4. Expose the port

* Rootless Podman / Docker: `sudo ufw allow 8085/tcp`
* Root Podman: published ports go through the FORWARD chain, so also run
  `sudo ufw route allow proto tcp to any port 8080`
* Cloud VM: add an inbound TCP 8085 rule to the security group / firewall.

## Troubleshooting: "Temporary failure in name resolution" during build (root Podman + ufw)

Symptoms: `npm ERR! EAI_AGAIN`, `Temporary failure resolving 'deb.debian.org'`,
`apk ... DNS: transient error`, `read udp 10.88.0.x -> <dns>:53: i/o timeout`.

This comes from the host firewall, not the Compose file. As root, containers sit on the Podman bridge
(`podman0`, 10.88.0.0/16). Their traffic to the internet is *routed* through the host, and Ubuntu's
ufw default is `deny (routed)`, so DNS gets dropped. Check with
`sudo journalctl -k | grep "UFW BLOCK" | grep podman`.

Fix it on the host with any one of:

* Run rootless (no `sudo`). Rootless networking does not go through ufw's FORWARD chain.
* Allow routed traffic and DNS from Podman bridges (one-time, per host):
  ```bash
  sudo ufw route allow in on podman0
  sudo ufw route allow in on podman1
  sudo ufw allow in on podman+ to any port 53
  ```
  (`podman1`, `podman2`, and so on are the bridges compose networks create. `podman network inspect` shows the name.)
* Build only, without changing ufw: `sudo podman-compose --podman-build-args=--network=host build`.

## Other portability notes

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
