# docker-dns-sync Releases

Публичный репозиторий релизов Debian-пакетов **docker-dns-sync**.

Исходный код проекта находится в приватном репозитории, а готовые бинарные пакеты (`.deb`) автоматически собираются с помощью GitHub Actions и публикуются здесь.

[![Releases](https://img.shields.io/github/v/release/Jon-Donovan/docker-dns-sync-releases)](https://github.com/Jon-Donovan/docker-dns-sync-releases/releases)
[![Platform](https://img.shields.io/badge/platform-Debian%20%7C%20Ubuntu-red.svg)](#)
[![Build](https://img.shields.io/badge/build-GitHub%20Actions-blue.svg)](#)

---

## Что такое docker-dns-sync?

`docker-dns-sync` автоматически синхронизирует DNS-имена запущенных Docker-контейнеров с `dnsmasq` через Docker Events.

Сервис создаёт записи вида:

```text
php74-fpm.docker
redis-server.docker
gitea-server-1.docker
```

Дополнительно можно задать пользовательское DNS-имя через label контейнера:

```yaml
labels:
  dns.hostname: alertmanager.docker
```

После событий Docker `rename`, `start`, `die`, `destroy`, `stop` и `restart` список DNS-записей пересобирается, а `dnsmasq` автоматически перезагружается.

---

## Возможности

- автоматическое создание записи `<container>.docker` для каждого запущенного контейнера;
- поддержка пользовательских DNS-алиасов через label `dns.hostname`;
- обновление записей в реальном времени через Docker Events;
- интеграция с `dnsmasq` через автоматически генерируемый hosts-файл;
- минималистичная реализация на Bash без дополнительных runtime-зависимостей;
- распространение в виде стандартного Debian-пакета.

---

## Установка

Скачайте последнюю версию пакета со страницы Releases:

```bash
wget https://github.com/Jon-Donovan/docker-dns-sync-releases/releases/latest/download/docker-dns-sync_<VERSION>_all.deb
```

Установите пакет:

```bash
sudo apt install ./docker-dns-sync_<VERSION>_all.deb
```

Либо:

```bash
sudo dpkg -i docker-dns-sync_<VERSION>_all.deb
sudo apt -f install
```

---

## Устанавливаемые файлы

Пакет устанавливает следующие файлы:

```text
/usr/bin/update-docker-dnsmasq-hosts
/usr/bin/docker-dnsmasq-watcher
/lib/systemd/system/docker-dns-sync.service
/etc/docker-dns-sync.conf
/etc/dnsmasq.d/docker-dns-sync.conf
/usr/share/doc/docker-dns-sync/README.md
```

Во время установки автоматически:

- создаётся каталог `/etc/dnsmasq.d/docker-hosts`;
- создаётся файл `containers.hosts`;
- выполняется `systemctl daemon-reload`;
- включается и запускается сервис `docker-dns-sync.service`.

---

## Конфигурация

Основной файл конфигурации:

```text
/etc/docker-dns-sync.conf
```

Значения по умолчанию:

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

Если файл `/etc/docker-dns-sync.conf` существует, он загружается перед построением DNS-записей. В противном случае используются встроенные значения по умолчанию.

---

## Конфигурация dnsmasq

Пакет устанавливает файл:

```text
/etc/dnsmasq.d/docker-dns-sync.conf
```

Содержимое:

```ini
addn-hosts=/etc/dnsmasq.d/docker-hosts/containers.hosts
local=/docker/
```

Сгенерированные записи хранятся в файле:

```text
/etc/dnsmasq.d/docker-hosts/containers.hosts
```

---

## Сервис systemd

Проверка состояния:

```bash
sudo systemctl status docker-dns-sync
```

Просмотр логов:

```bash
journalctl -u docker-dns-sync -f
```

Перезапуск сервиса:

```bash
sudo systemctl restart docker-dns-sync
```

---

## Принцип работы

### update-docker-dnsmasq-hosts

Скрипт выполняет следующие действия:

1. Загружает `/etc/docker-dns-sync.conf`, если файл существует.
2. Получает список работающих контейнеров через `docker ps`.
3. Извлекает первый доступный IP-адрес с помощью `docker inspect`.
4. Создаёт запись вида `<container>.$DNS_SUFFIX`.
5. Добавляет дополнительную запись, если присутствует label `dns.hostname`.
6. Сортирует и сохраняет уникальные записи в `HOSTS_FILE`.
7. Выполняет перезагрузку `dnsmasq`.

Пример результата:

```text
172.19.0.2 gitea-server-1.docker
172.20.0.2 php74-fpm.docker
172.23.0.3 redis-server.docker
```

### docker-dnsmasq-watcher

Наблюдатель:

- выполняет начальное построение DNS-записей;
- подписывается на Docker Events;
- отслеживает события:

  - `rename`
  - `start`
  - `die`
  - `destroy`
  - `stop`
  - `restart`

- после каждого события повторно генерирует hosts-файл.

---

## Пользовательские DNS-алиасы

Пример `docker-compose.yml`:

```yaml
services:
  alertmanager:
    image: prom/alertmanager
    labels:
      dns.hostname: alertmanager.docker
```

Будут созданы две записи:

```text
172.21.0.3 alertmanager-container-name.docker
172.21.0.3 alertmanager.docker
```

---

## Проверка работы

Проверить разрешение имени можно командами:

```bash
getent hosts redis-server.docker
```

или:

```bash
dig @127.0.0.1 redis-server.docker
```

---

## Процесс публикации релизов

Данный репозиторий содержит только автоматически собранные бинарные пакеты.

Приватный репозиторий исходного кода:

- собирает Debian-пакеты с помощью GitHub Actions;
- публикует готовые `.deb`-артефакты в этот публичный репозиторий;
- сохраняет историю разработки и исходный код закрытыми.

---

## Лицензия

Информация о лицензии поставляется вместе с дистрибутивным пакетом.
