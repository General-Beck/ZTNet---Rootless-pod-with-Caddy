# Installing ZTNet in user-mode using podman rootless pod. EN | [RU](README_RU.MD)

---

##Overview

* **Dependencies**: Only `podman` and `passt` are installed to get rootless containerization and user-mode networking without additional daemons.
* **Why `pasta`?**: In a rootless environment, Podman uses `pasta` (a binary from the `passt` package) instead of CNI/Netavark, which provides native networking without NAT and without privileges.
* **systemd-user stack**: five `.service` units in `~/.config/systemd/user`: a pod (`ztnet-pod.service`) and four containers (`postgres`, `zerotier`, `ztnet`, `caddy`).
* **Auto-update**: via `podman-auto-update.timer` with `registry` policy.
* **linger**: enable `loginctl Enable-Linger` for automatic startup without login.

---

## 1. Installing dependencies

``` bash
sudo apt update
sudo apt install -y podman passt
```

* `podman` is specified in the official Ubuntu 24.04 repositories and is configured via APT, without the need for a PPA.
* `passt` (the `pasta` command appears when installed on the system) implements a user-mode network driver that does not require privileges.

---

## 2. Why `pasta`?

1. **No NAT** and copying host IP into container namespace - gives container the same address and routes as host ([Podman Documentation][3](https://docs.podman.io/en/stable/markdown/podman-network.1.html)).
2. **Unprivileged**: `pasta` creates TAP interface inside namespace and proxies TCP/UDP/ICMP socket surface of host ([Ubuntu Manpages][4](https://manpages.ubuntu.com/manpages/noble/man1/pasta.1.html)).
3. **High performance** for rootless containers compared to `slirp4netns` ([Oracle Documentation][5](https://docs.oracle.com/en/learn/ol-podman-pasta-networking/)).

By default, non-root Podman uses the Netavark backend, but the actual user-mode network is provided by `pasta` (from the `passt` package) ([Podman Documentation][3](https://docs.podman.io/en/stable/markdown/podman-network.1.html)).

---

## 3. Preparing the `/opt/ztnet-data` directory

``` bash
sudo mkdir -p /opt/ztnet-data/postgres-data \
/opt/ztnet-data/zerotier \
/opt/ztnet-data/caddy_data
sudo chown -R ztnet:ztnet /opt/ztnet-data
```

These folders **must** be created in advance so that Podman does not return an error about the directory not existing when mounting `-v /opt/ztnet-data/...`.

--

## 4. Contents of services

All files are located in `~/.config/systemd/user/`.

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
--name ztnet-pod\
--network pasta \
--add-host=zerotier:127.0.0.1 \
-p 80:80\
-p 443:443\
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
[Module]
Description=PostgreSQL Container
Documentation=man:podman-generate-systemd(1)
Part=ztnet-pod.service
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
--health-interval 30s\
--health-on-failure restart \
--health-retries 3\
--pod=ztnet-pod\
docker.io/library/postgres:15.8-alpine
KillMode=mixed
ExecStop=/usr/bin/podman rm -v -f -i --cidfile=%t/%N.cid
ExecStopPost=-/usr/bin/podman rm -v -f -i --cidfile=%t/%N.cid
Delegation=yes
Type=notification
Notification Access=all
Syslog Identifier=%N

[Install]
WantedBy=default.target
```

### 4.3 `container-zerotier.service`

```
[Module]
Description=ZeroTier Container
Documentation=man:podman-generate-systemd(1)
Part=ztnet-pod.service
Requires=network-online.target
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
--replace\ 
--log-driver journald\ 
--sdnotify=conmon \ 
-d\ 
--device=/dev/net/tun \ 
--cap-add=NET_ADMIN,SYS_ADMIN \ 
-v /opt/ztnet-data/zerotier:/var/lib/zerotier-one \ 
--env ZT_ALLOW_MANAGEMENT_FROM=127.0.0.1 \ 
--env ZT_OVERRIDE_LOCAL_CONF=true \ 
--label io.containers.autoupdate=registry \ 
--health-cmd "zerotier-cli info" \ 
--health-interval 30s\ 
--health-on-failure restart \ 
--health-retries 3\ 
--pod=ztnet-pod\ 
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
--name=ztnet\ 
--cidfile=%t/%N.cid \ 
--cgroups=split \ 
--replace\ 
--log-driver journald\ 
--sdnotify=conmon \ 
-d\ 
-v /opt/ztnet-data/zerotier:/var/lib/zerotier-one \ 
--env HOSTNAME=0.0.0.0 \ 
--env NEXTAUTH_SECRET='pandom_secret' \ 
--env NEXTAUTH_URL=https://you.domain\ 
--env NEXTAUTH_URL_INTERNAL=https://you.domain \ 
--env POSTGRES_DB=ztnet \ 
--env POSTGRES_HOST=localhost \ 
--env POSTGRES_PASSWORD='pg_password' \ 
--env POSTGRES_PORT=5432 \ 
--env POSTGRES_USER=postgres \ 
--label io.containers.autoupdate=registry \ 
--health-cmd "curl -f http://ztnet-pod:3000/api/auth/session || exit 1" \ 
--health-interval 30s\ 
--health-on-failure restart \ 
--health-retries 3\ 
--health-start-period 30s\ 
--pod=ztnet-pod\ 
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
--name=caddy\ 
--cidfile=%t/%N.cid \ 
--cgroups=split \ 
--replace\ 
--log-driver journald\ 
--sdnotify=conmon \ 
-d\ 
-v /opt/ztnet-data/caddy_data:/data\ 
--label io.containers.autoupdate=registry \ 
--health-cmd "nc -z localhost 80 || exit 1"\ 
--health-interval 30s\ 
--health-on-failure restart \ 
--health-retries 3\ 
--health-timeout 10s\ 
--pod=ztnet-pod\ 
docker.io/library/caddy:latest\ 
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

## 5. Deployment and verification

### 5.1 Root (one-time)

```bash
sudo loginctl enable-linger ztnet
```

– enable linger so that user units are launched without login ([freedesktop.org][6](https://www.freedesktop.org/software/systemd/man/loginctl.html)).

### 5.2 User `ztnet` (rootless)

```bash
# 1) Reload user-systemd daemons:
systemctl --user daemon-reload

# 2) Start pod:
systemctl --user enable --now ztnet-pod.service
systemctl --user status ztnet-pod.service

# 3) Start containers:
systemctl --user enable --now \
container-postgres.service \
container-zerotier.service \
container-ztnet.service \
container-caddy.service

# 4) Check statuses:
podman ps --format '{{.Names}}\t{{.Status}}'

# 5) Enable auto-update:
systemctl --user enable --now podman-auto-update.timer
podman auto-update --dry-run
```

---

## 6. Final test (root)

```bash
sudo reboot

# After booting, as root:
systemctl --user --machine=ztnet@ status ztnet-pod.service
podman --remote --url=unix:/run/user/1000/podman/podman.sock ps
```

As a result, all services start automatically in the `ztnet` session, work rootless, update and self-heal thanks to health-checks and RestartPolicy.

[1]: https://docs.vultr.com/how-to-install-podman-on-ubuntu-24-04 "How to Install Podman on Ubuntu 24.04 - Vultr Docs"
[2]: https://launchpad.net/ubuntu/noble/%2Bpackage/passt "passt : Noble (24.04) : Ubuntu - Launchpad"
[3]: https://docs.podman.io/en/stable/markdown/podman-network.1.html "podman-network"
[4]: https://manpages.ubuntu.com/manpages/noble/man1/pasta.1.html "passt - Unprivileged user-mode network connectivity for virtual ..."
[5]: https://docs.oracle.com/en/learn/ol-podman-pasta-networking/ "Use Pasta Networking with Podman on Oracle Linux"
[6]: https://www.freedesktop.org/software/systemd/man/loginctl.html "loginctl - Freedesktop.org"
