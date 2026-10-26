# syslogserver (Rsyslogd + PimpMyLog)

This repository only contains the **syslog server** container: rsyslogd receives
syslog over UDP/514 and the logs are browsable through the
[PimpMyLog](http://pimpmylog.com) web UI (nginx + php-fpm, supervised by supervisord).

## Run

    docker build -t syslogserver ./syslogserver
    docker run -d --name syslogserver \
      -e SYSLOG_USERNAME=admin \
      -e SYSLOG_PASSWORD=1234 \
      -p 8080:80 -p 514:514/udp \
      -v /path/to/logs:/var/log/net \
      syslogserver

Credentials for the PimpMyLog UI are created from `SYSLOG_USERNAME` / `SYSLOG_PASSWORD`
(see `syslogserver/create-user.php`). Point your devices at `<host>:514/udp`.

See [syslogserver/README.md](syslogserver/README.md) for details.
