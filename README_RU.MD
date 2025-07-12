# Установка [ZTNet](https://ztnet.network/) в окружении пользователя с использованием podman rootless pod. RU | [EN](README.MD)

---

## Краткий обзор

* **Зависимости**: устанавливаем только `podman` и `passt`, чтобы получить rootless-контейнеризацию и user-mode networking без дополнительных демонов.
* **Почему `pasta`?**: в rootless-среде Podman использует `pasta` (бинарник из пакета `passt`) вместо CNI/Netavark, что обеспечивает нативное сетевое соединение без NAT и без привилегий.
* **Стек systemd-user**: пять `.service`-юнитов в `~/.config/systemd/user`: под (`ztnet-pod.service`) и четыре контейнера (`postgres`, `zerotier`, `ztnet`, `caddy`).
* **Авто-обновление**: через `podman-auto-update.timer` с политикой `registry`.
* **linger**: включаем `loginctl enable-linger` для автоматического старта без входа.

---

## 1. Установка зависимостей

```bash
sudo apt update
sudo apt install -y podman passt
```

* `podman` поставляется в официальных репозиториях Ubuntu 24.04 и устанавливается через APT, без необходимости PPAs.
* `passt` (при установке в систему появляется команда `pasta`) реализует user-mode сетевой драйвер, не требующий привилегий.

---

## 2. Зачем `pasta`?

1. **Без NAT** и с копированием IP-хоста в namespace контейнера — контейнер получает тот же адрес и маршруты, что и хост ([Podman Documentation][1](https://docs.podman.io/en/stable/markdown/podman-network.1.html)).
2. **Работает без привилегий**: `pasta` создаёт TAP-интерфейс внутри namespace и проксирует TCP/UDP/ICMP поверх сокетов хоста ([Ubuntu Manpages][2](https://manpages.ubuntu.com/manpages/noble/man1/pasta.1.html)).
3. **Высокая производительность** для rootless-контейнеров по сравнению со `slirp4netns` ([Oracle Documentation][3](https://docs.oracle.com/en/learn/ol-podman-pasta-networking/)).

По умолчанию в rootless Podman выбран backend Netavark, но реальную user-mode сеть обеспечивает `pasta` (из пакета `passt`).

---

## 3. Подготовка директорий `/opt/ztnet-data`

```bash
sudo mkdir -p /opt/ztnet-data/postgres-data \
             /opt/ztnet-data/zerotier \
             /opt/ztnet-data/caddy_data
sudo chown -R ztnet:ztnet /opt/ztnet-data
```

Эти папки **необходимо** создать заранее, чтобы Podman при монтировании `-v /opt/ztnet-data/...` не выдал ошибку об отсутствии каталога.

---

## 4. Содержимое сервисов

Все файлы лежат в `~/.config/systemd/user/`.

### 4.1 `ztnet-pod.service`

```
[Unit]
Description=ZTNet Pod
Documentation=man:podman-generate-systemd(1)
Wants=network-online.target
After=network-online.target
Wants=container-postgres.service container-zerotier.service container-ztnet.service container-caddy.service
Before=container-postgres.service container-zerotier.service container-ztnet.service container-caddy.service

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=on-failure
TimeoutStopSec=70
ExecStartPre=-/usr/bin/podman pod create \
        --name ztnet-pod \
        --network pasta \
        --add-host=zerotier:127.0.0.1 \
        -p 80:80 \
        -p 443:443 \
        -p 9993:9993/udp
ExecStart=/usr/bin/podman pod start ztnet-pod
ExecStop=/usr/bin/podman pod stop -t 60 ztnet-pod
TimeoutStopSec=60
Type=oneshot
RemainAfterExit=yes

[Install]
WantedBy=default.target
```

### 4.2 `container-postgres.service`

```
[Unit]
Description=PostgreSQL Container
Documentation=man:podman-generate-systemd(1)
PartOf=ztnet-pod.service
Wants=network-online.target
Requires=ztnet-pod.service
After=ztnet-pod.service network-online.target
RequiresMountsFor=/opt/ztnet-data/postgres-data

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=on-failure
TimeoutStopSec=70
ExecStart=/usr/bin/podman run \
        --name=postgres \
        --cidfile=%t/%N.cid \
        --cgroups=split \
        --replace \
        --log-driver journald \
        --sdnotify=conmon \
        -d \
        -v /opt/ztnet-data/postgres-data:/var/lib/postgresql/data \
        --env POSTGRES_DB=ztnet \
        --env POSTGRES_PASSWORD='pq_password' \
        --env POSTGRES_USER=postgres \
        --label io.containers.autoupdate=registry \
        --health-cmd "pg_isready -U postgres -d ztnet" \
        --health-interval 30s \
        --health-on-failure restart \
        --health-retries 3 \
        --pod=ztnet-pod \
        docker.io/library/postgres:15.8-alpine
KillMode=mixed
ExecStop=/usr/bin/podman rm -v -f -i --cidfile=%t/%N.cid
ExecStopPost=-/usr/bin/podman rm -v -f -i --cidfile=%t/%N.cid
Delegate=yes
Type=notify
NotifyAccess=all
SyslogIdentifier=%N

[Install]
WantedBy=default.target
```

### 4.3 `container-zerotier.service`

```
[Unit]
Description=ZeroTier Container
Documentation=man:podman-generate-systemd(1)
PartOf=ztnet-pod.service
Wants=network-online.target
Requires=ztnet-pod.service
After=ztnet-pod.service
RequiresMountsFor=/opt/ztnet-data/zerotier

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=on-failure
TimeoutStopSec=70
ExecStart=/usr/bin/podman run \
        --name=zerotier \
        --cidfile=%t/%N.cid \
        --cgroups=split \
        --replace \
        --log-driver journald \
        --sdnotify=conmon \
        -d \
        --device=/dev/net/tun \
        --cap-add=NET_ADMIN,SYS_ADMIN \
        -v /opt/ztnet-data/zerotier:/var/lib/zerotier-one \
        --env ZT_ALLOW_MANAGEMENT_FROM=127.0.0.1 \
        --env ZT_OVERRIDE_LOCAL_CONF=true \
        --label io.containers.autoupdate=registry \
        --health-cmd "zerotier-cli info" \
        --health-interval 30s \
        --health-on-failure restart \
        --health-retries 3 \
        --pod=ztnet-pod \
        docker.io/zyclonite/zerotier:latest
KillMode=mixed
ExecStop=/usr/bin/podman rm -v -f -i --cidfile=%t/%N.cid
ExecStopPost=-/usr/bin/podman rm -v -f -i --cidfile=%t/%N.cid
Delegate=yes
Type=notify
NotifyAccess=all
SyslogIdentifier=%N

[Install]
WantedBy=default.target
```

### 4.4 `container-ztnet.service`

```
[Unit]
Description=ZTNet Web UI Container
Documentation=man:podman-generate-systemd(1)
PartOf=ztnet-pod.service
After=ztnet-pod.service container-postgres.service container-zerotier.service
Requires=ztnet-pod.service container-postgres.service container-zerotier.service
RequiresMountsFor=/opt/ztnet-data/zerotier

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=on-failure
TimeoutStopSec=70
ExecStart=/usr/bin/podman run \
        --name=ztnet \
        --cidfile=%t/%N.cid \
        --cgroups=split \
        --replace \
        --log-driver journald \
        --sdnotify=conmon \
        -d \
        -v /opt/ztnet-data/zerotier:/var/lib/zerotier-one \
        --env HOSTNAME=0.0.0.0 \
        --env NEXTAUTH_SECRET='pandom_secret' \
        --env NEXTAUTH_URL=https://you.domain \
        --env NEXTAUTH_URL_INTERNAL=https://you.domain \
        --env POSTGRES_DB=ztnet \
        --env POSTGRES_HOST=localhost \
        --env POSTGRES_PASSWORD='pg_password' \
        --env POSTGRES_PORT=5432 \
        --env POSTGRES_USER=postgres \
        --label io.containers.autoupdate=registry \
        --health-cmd "curl -f http://ztnet-pod:3000/api/auth/session || exit 1" \
        --health-interval 30s \
        --health-on-failure restart \
        --health-retries 3 \
        --health-start-period 30s \
        --pod=ztnet-pod \
        docker.io/sinamics/ztnet:latest
KillMode=mixed
ExecStop=/usr/bin/podman rm -v -f -i --cidfile=%t/%N.cid
ExecStopPost=-/usr/bin/podman rm -v -f -i --cidfile=%t/%N.cid
Delegate=yes
Type=notify
NotifyAccess=all
SyslogIdentifier=%N

[Install]
WantedBy=default.target
```

### 4.5 `container-caddy.service`

```
[Unit]
Description=Caddy Reverse Proxy
Documentation=man:podman-generate-systemd(1)
PartOf=ztnet-pod.service
Wants=network-online.target
Requires=ztnet-pod.service container-ztnet.service container-zerotier.service container-postgres.service
After=ztnet-pod.service container-ztnet.service container-postgres.service container-zerotier.service
RequiresMountsFor=/opt/ztnet-data/caddy_data

[Service]
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=on-failure
TimeoutStopSec=70
ExecStart=/usr/bin/podman run \
        --name=caddy \
        --cidfile=%t/%N.cid \
        --cgroups=split \
        --replace \
        --log-driver journald \
        --sdnotify=conmon \
        -d \
        -v /opt/ztnet-data/caddy_data:/data \
        --label io.containers.autoupdate=registry \
        --health-cmd "nc -z localhost 80 || exit 1" \
        --health-interval 30s \
        --health-on-failure restart \
        --health-retries 3 \
        --health-timeout 10s \
        --pod=ztnet-pod \
        docker.io/library/caddy:latest \
        caddy reverse-proxy --from you.domain --to ztnet-pod:3000
KillMode=mixed
ExecStop=/usr/bin/podman rm -v -f -i --cidfile=%t/%N.cid
ExecStopPost=-/usr/bin/podman rm -v -f -i --cidfile=%t/%N.cid
Delegate=yes
Type=notify
NotifyAccess=all
SyslogIdentifier=%N

[Install]
WantedBy=default.target
```

---

## 5. Развёртывание и проверка

### 5.1 Root (единожды)

```bash
sudo loginctl enable-linger ztnet
```

– включаем linger, чтобы user-юниты запускались без логина ([freedesktop.org][4]).

### 5.2 Пользователь `ztnet` (rootless)

```bash
# 1) Перезагрузить демоны user-systemd:
systemctl --user daemon-reload

# 2) Запустить pod:
systemctl --user enable --now ztnet-pod.service
systemctl --user status ztnet-pod.service

# 3) Запустить контейнеры:
systemctl --user enable --now \
  container-postgres.service \
  container-zerotier.service \
  container-ztnet.service \
  container-caddy.service

# 4) Проверить статусы:
podman ps --format '{{.Names}}\t{{.Status}}'

# 5) Включить авто-обновление:
systemctl --user enable --now podman-auto-update.timer
podman auto-update --dry-run
```

---

## 6. Финальный тест (root)

```bash
sudo reboot

# После загрузки, как root:
systemctl --user --machine=ztnet@ status ztnet-pod.service
podman --remote --url=unix:/run/user/1000/podman/podman.sock ps
```

В результате все сервисы стартуют автоматически в сессии `ztnet`, работают rootless, обновляются и self-heal благодаря health-checks и RestartPolicy.

[1]: https://docs.podman.io/en/stable/markdown/podman-network.1.html "podman-network"
[2]: https://manpages.ubuntu.com/manpages/noble/man1/pasta.1.html "passt - Unprivileged user-mode network connectivity for virtual ..."
[3]: https://docs.oracle.com/en/learn/ol-podman-pasta-networking/ "Use Pasta Networking with Podman on Oracle Linux"
[4]: https://www.freedesktop.org/software/systemd/man/loginctl.html "loginctl - Freedesktop.org"
