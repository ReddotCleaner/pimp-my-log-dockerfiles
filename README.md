# syslogserver — Rsyslog + PimpMyLog in one container

A self-contained remote syslog server: **rsyslogd** receives syslog over **UDP 514 and TCP 514**
and **PimpMyLog** gives you a web UI to browse, search and export the collected logs.

Built on Ubuntu 24.04 + PHP 8.3, published to GHCR by GitHub Actions, and smoke-tested on every push
(UDP + TCP ingest, web UI, login, log parsing).

[![Build syslogserver image](https://github.com/ReddotCleaner/pimp-my-log-dockerfiles/actions/workflows/docker-build.yml/badge.svg)](https://github.com/ReddotCleaner/pimp-my-log-dockerfiles/actions/workflows/docker-build.yml)

## Why this fork

This repo was forked from [pbertera/dockerfiles](https://github.com/pbertera/dockerfiles) and reduced to a
single container. The original image was Ubuntu 14.04 + PHP 5 and **only listened on UDP 514** — a TCP client
(Windows + NXLog, `logger -n host -P 514 -T`, etc.) got `Connection refused`. This version:

| | before | now |
|---|---|---|
| base image | `32bit/ubuntu:14.04` (EOL) | `ubuntu:24.04` |
| PHP | 5 (`php5-fpm`) | 8.3 (`php8.3-fpm`) |
| listeners | `514/udp` | `514/udp` **+ `514/tcp`** |
| build | manual | GitHub Actions → GHCR + smoke tests |

Getting PimpMyLog (last commit 2017) to run on PHP 8 required a real patch — see [Notes](#notes).

## Quick start

```bash
docker run -d --name syslogserver --restart unless-stopped \
  -e SYSLOG_USERNAME=admin \
  -e SYSLOG_PASSWORD='change-me' \
  -p 514:514/udp \
  -p 514:514/tcp \
  -p 8080:80 \
  ghcr.io/reddotcleaner/pimp-my-log-dockerfiles:latest
```

Then open `http://<host>:8080` and sign in with the credentials above.

> **Both port mappings matter.** `-p 514:514/tcp` is what the original image was missing — leave it out and
> TCP clients are refused again. Host port 514 needs root (or `sudo docker run`); if something else already
> owns 514, map another host port instead, e.g. `-p 1514:514/tcp -p 1514:514/udp`.

The image is **publicly pullable** — no `docker login` needed:

```
ghcr.io/reddotcleaner/pimp-my-log-dockerfiles:latest
ghcr.io/reddotcleaner/pimp-my-log-dockerfiles:sha-<commit>
```

## Configuration

| env var | default | notes |
|---|---|---|
| `SYSLOG_USERNAME` | `admin` | PimpMyLog web login (and the `/syslog.log` basic-auth user) |
| `SYSLOG_PASSWORD` | `SyslogP4ss` | **first start only** — the user is written to `config.auth.user.php`; changing the env var later does not change an existing user. Minimum 6 characters. |

| path | notes |
|---|---|
| `/var/log/net/syslog.log` | the collected log file |
| `/var/log/net` | mount a volume here to persist logs: `-v /srv/syslog:/var/log/net` |
| `/var/www` | PimpMyLog (cloned from [potsky/PimpMyLog](https://github.com/potsky/PimpMyLog)) |

If you bind-mount a host directory on `/var/log/net`, the container writes as **uid 102 / gid 4**
(`syslog:adm`), so `chown -R 102:4 /srv/syslog` on the host or rsyslog will not be able to write.

A mounted `/var/log/net` also means logs survive container re-creation — recommended for a log server.

## Sending logs

Log format written by the container (PimpMyLog parses all four columns):

```
27-Oct-2026 02:10:11.500 192.168.31.71 info  <message>
```

**Linux (rsyslog client)** — `/etc/rsyslog.d/50-remote.conf`:

```
*.* @@192.168.31.5:514     # TCP
*.* @192.168.31.5:514      # UDP
```

**Windows (NXLog)** — `nxlog.conf`:

```
<Output out>
    Module      om_tcp
    Host        192.168.31.5
    Port        514
    Exec        $Message = $raw_event;
</Output>
```

NXLog only forwards events created **after** it starts — old crash logs are not replayed.

**Quick manual test** (no client needed):

```bash
printf '<14>Oct 27 05:00:00 myhost test: hello over TCP\n' > /dev/tcp/192.168.31.5/514
printf '<14>Oct 27 05:00:01 myhost test: hello over UDP\n' | nc -u -w1 192.168.31.5 514
```

## Build it yourself

```bash
git clone https://github.com/ReddotCleaner/pimp-my-log-dockerfiles.git
cd pimp-my-log-dockerfiles
docker build -t syslogserver ./syslogserver
docker run -d --name syslogserver -e SYSLOG_USERNAME=admin -e SYSLOG_PASSWORD=test1234 \
  -p 8080:80 -p 1514:514/tcp -p 1514:514/udp syslogserver
```

Pushing to `master` (or a manual `workflow_dispatch`) triggers
[`.github/workflows/docker-build.yml`](.github/workflows/docker-build.yml): build → push to GHCR →
start the image and assert UDP 514 + TCP 514 listeners, ingest over both transports, web UI `HTTP 200`,
login `302`, and that PimpMyLog actually parses the ingested lines (`count > 0`).

## Files

| file | purpose |
|---|---|
| `syslogserver/Dockerfile` | image build (Ubuntu 24.04, nginx + php8.3-fpm + rsyslog + supervisord) |
| `syslogserver/rsyslog.conf` | `imudp` + `imtcp` on 514, custom `customdate` template, `*.* /var/log/net/syslog.log` |
| `syslogserver/nginx-default` | nginx site; PHP via `/run/php/php8.3-fpm.sock`, `/syslog.log` behind basic auth |
| `syslogserver/supervisord.conf` | runs rsyslogd, php-fpm and nginx in one container |
| `syslogserver/run.sh` | creates the htpasswd + PimpMyLog user on first start, then starts supervisord |
| `syslogserver/config.user.php` | PimpMyLog log definition (path + parsing regex) |
| `syslogserver/create-user.php` | creates the PimpMyLog user from `SYSLOG_USERNAME`/`SYSLOG_PASSWORD` |

## Notes

- **PimpMyLog on PHP 8.** PimpMyLog 1.7.14 (2017) calls `get_magic_quotes_gpc()` and `each()` — both removed
  in PHP 8.0, both a fatal error on PHP 8.3. They live in one dead `if ( get_magic_quotes_gpc() )` block, so the
  Dockerfile guards that block with `false`; no live code path is touched. Built with `php -l` assertions.
- **`variables_order` must include `E`.** `create-user.php` reads `$_ENV`, so the image sets
  `variables_order = "GPCSE"`. Forget it and no user is created — the web UI then rejects every login.
- **`fastcgi_params` has no `SCRIPT_FILENAME`** on Debian/Ubuntu (only `fastcgi.conf` does). The nginx site
  passes it explicitly, otherwise php-fpm cannot find the script and nginx answers 502.
- **Timezone.** `config.user.php` ships with `USER_TIME_ZONE = Europe/Paris` and the container clock is UTC.
  If your timestamps look off, change it to e.g. `Asia/Shanghai` — or switch it in the UI (Settings → timezone).
- **Rate of refresh / line count** for the "Syslog" view are set in `config.user.php` (`refresh: 20`, `max: 20`).
- **Plain HTTP only.** The UI has no TLS and the container has no HTTPS listener; put it behind a reverse proxy
  if it needs to be reachable from the internet.
- **Legacy rsyslog syntax** (`$ModLoad`, `$InputTCPServerRun`) is used on purpose — it is still supported by
  rsyslog 8.2312 and keeps the file diffable against upstream.

## Credits & license

- Container packaging derived from [pbertera/dockerfiles](https://github.com/pbertera/dockerfiles).
- [PimpMyLog](https://github.com/potsky/PimpMyLog) by Potsky, GPLv3 — fetched from GitHub **at build time**,
  not vendored here.

---

## 中文说明

**这是什么**：一个容器里同时跑 rsyslogd（收日志）和 PimpMyLog（网页看日志）。相比上游老镜像，主要修了两件事：
基础镜像从 Ubuntu 14.04/PHP 5 升到 **24.04/PHP 8.3**；rsyslog 除了 UDP 514 还**监听 TCP 514**（老镜像只有 UDP，
所以 Windows/NXLog 走 TCP 会被 `Connection refused`）。

**起服务**：

```bash
docker run -d --name syslogserver --restart unless-stopped \
  -e SYSLOG_USERNAME=admin -e SYSLOG_PASSWORD='你的密码' \
  -p 514:514/udp -p 514:514/tcp -p 8080:80 \
  -v /srv/syslog:/var/log/net \
  ghcr.io/reddotcleaner/pimp-my-log-dockerfiles:latest
```

**要点**：

- `-p 514:514/tcp` **必须显式加**，否则又回到 TCP 被拒的老问题；宿主 514 需要 root，被占用时改用 `-p 1514:514/tcp`。
- 挂载 `/var/log/net` 时，宿主目录要给容器内的 `syslog:adm`（**uid 102 / gid 4**）：`chown -R 102:4 /srv/syslog`。
- `SYSLOG_PASSWORD` **只在第一次启动生效**（用户写进 `config.auth.user.php`），之后改环境变量不会改密码；密码至少 6 位。
- 日志时间看起来偏 8 小时，是 `config.user.php` 里 `USER_TIME_ZONE` 默认 `Europe/Paris`，改成 `Asia/Shanghai` 即可。
- 镜像公开可拉，不需要 `docker login`。

**自测**：

```bash
printf '<14>Oct 27 05:00:00 myhost test: hello over TCP\n' > /dev/tcp/127.0.0.1/514
docker exec syslogserver cat /var/log/net/syslog.log
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/    # 期望 200
```
