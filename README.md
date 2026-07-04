# docker-dns-sync Releases

Public release repository for **docker-dns-sync** Debian packages.

The source code repository is private, but binary releases (`.deb`) are built automatically using GitHub Actions and published here.

[![Releases](https://img.shields.io/github/v/release/Jon-Donovan/docker-dns-sync-releases)](https://github.com/Jon-Donovan/docker-dns-sync-releases/releases)
[![Platform](https://img.shields.io/badge/platform-Debian%20%7C%20Ubuntu-red.svg)](#)
[![Build](https://img.shields.io/badge/build-GitHub%20Actions-blue.svg)](#)

---

## What is docker-dns-sync?

`docker-dns-sync` automatically synchronizes DNS names of running Docker containers with `dnsmasq` using Docker Events.

The service creates DNS records such as:

```text
php74-fpm.docker
redis-server.docker
gitea-server-1.docker
```

Additional DNS names may be defined using a Docker label:

```yaml
labels:
  dns.hostname: alertmanager.docker
```

Whenever Docker emits `rename`, `start`, `die`, `destroy`, `stop`, or `restart` events, the generated hosts file is rebuilt and `dnsmasq` is reloaded automatically.

---

## Features

- automatic `<container>.docker` DNS records for running containers;
- optional custom DNS aliases via `dns.hostname` labels;
- real-time updates using Docker Events;
- integration with `dnsmasq` through generated hosts files;
- lightweight Bash implementation with no additional runtime dependencies;
- distributed as a standard Debian package.

---

## Installation

Download the latest package from the Releases page:

```bash
wget https://github.com/Jon-Donovan/docker-dns-sync-releases/releases/latest/download/docker-dns-sync_<VERSION>_all.deb
```

Install it:

```bash
sudo apt install ./docker-dns-sync_<VERSION>_all.deb
```

Alternatively:

```bash
sudo dpkg -i docker-dns-sync_<VERSION>_all.deb
sudo apt -f install
```

---

## Installed Files

The package installs:

```text
/usr/bin/update-docker-dnsmasq-hosts
/usr/bin/docker-dnsmasq-watcher
/lib/systemd/system/docker-dns-sync.service
/etc/docker-dns-sync.conf
/etc/dnsmasq.d/docker-dns-sync.conf
/usr/share/doc/docker-dns-sync/README.md
```

During installation:

- `/etc/dnsmasq.d/docker-hosts` is created;
- `containers.hosts` is initialized;
- `systemd` is reloaded;
- `docker-dns-sync.service` is enabled and started automatically.

---

## Configuration

Main configuration file:

```text
/etc/docker-dns-sync.conf
```

Default values:

```bash
# DNS suffix for automatically generated container names
DNS_SUFFIX=docker

# Generated hosts file consumed by dnsmasq
HOSTS_FILE=/etc/dnsmasq.d/docker-hosts/containers.hosts

# dnsmasq systemd service name
DNSMASQ_SERVICE=dnsmasq

# Docker label used for custom DNS hostname
DNS_HOSTNAME_LABEL=dns.hostname
```

If `/etc/docker-dns-sync.conf` exists, it is loaded before rebuilding DNS records. Otherwise, built-in defaults are used.

---

## dnsmasq Configuration

The package installs:

```text
/etc/dnsmasq.d/docker-dns-sync.conf
```

Contents:

```ini
addn-hosts=/etc/dnsmasq.d/docker-hosts/containers.hosts
local=/docker/
```

The generated hosts file is stored at:

```text
/etc/dnsmasq.d/docker-hosts/containers.hosts
```

---

## systemd Service

Check status:

```bash
sudo systemctl status docker-dns-sync
```

View logs:

```bash
journalctl -u docker-dns-sync -f
```

Restart the service:

```bash
sudo systemctl restart docker-dns-sync
```

---

## How It Works

### `update-docker-dnsmasq-hosts`

The update script performs the following steps:

1. Loads `/etc/docker-dns-sync.conf` if it exists.
2. Retrieves the list of running containers using `docker ps`.
3. Obtains the first available IP address from `docker inspect`.
4. Creates an automatic DNS record `<container>.$DNS_SUFFIX`.
5. Adds an additional record when the `dns.hostname` label is present.
6. Writes sorted, unique entries to `HOSTS_FILE`.
7. Reloads the `dnsmasq` service.

Example output:

```text
172.19.0.2 gitea-server-1.docker
172.20.0.2 php74-fpm.docker
172.23.0.3 redis-server.docker
```

### `docker-dnsmasq-watcher`

The watcher:

- performs an initial DNS rebuild;
- subscribes to Docker Events;
- listens for:

  - `rename`
  - `start`
  - `die`
  - `destroy`
  - `stop`
  - `restart`

- regenerates DNS records after each event.

---

## Custom DNS Aliases

Example `docker-compose.yml`:

```yaml
services:
  alertmanager:
    image: prom/alertmanager
    labels:
      dns.hostname: alertmanager.docker
```

Generated records:

```text
172.21.0.3 alertmanager-container-name.docker
172.21.0.3 alertmanager.docker
```

---

## Verification

Verify that DNS resolution works correctly:

```bash
getent hosts redis-server.docker
```

```bash
dig @127.0.0.1 redis-server.docker
```

---

## Release Process

This repository contains only automatically generated binary releases.

The private source repository:

- builds Debian packages using GitHub Actions;
- publishes `.deb` artifacts to this public repository;
- keeps all implementation details and development history private.

---

## License

See the license information included in the distributed package.
