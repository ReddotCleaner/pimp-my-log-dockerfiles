FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update \
    && apt-get install -y git net-tools vim nginx rsyslog supervisor php8.3-fpm php8.3-cli apache2-utils \
    && rm -rf /var/lib/apt/lists/*

RUN sed -i -e 's/listen\ =\ 127.0.0.1:9000/listen\ =\ \/run\/php\/php8.3-fpm.sock/' /etc/php/8.3/fpm/pool.d/www.conf
RUN sed -i '1idaemon off;' /etc/nginx/nginx.conf

RUN rm -rf /var/www && git clone https://github.com/potsky/PimpMyLog.git /var/www
RUN sed -i 's/;daemonize\ =\ yes/daemonize\ =\ no/' /etc/php/8.3/fpm/php-fpm.conf
# create-user.php reads $_ENV, so E must be part of variables_order
# (upstream wrote \"GPCSE\" inside single quotes, which puts literal backslashes
# into php.ini and leaves $_ENV empty)
RUN sed -i 's/^variables_order =.*/variables_order = "GPCSE"/' /etc/php/8.3/cli/php.ini \
    && sed -i 's/^variables_order =.*/variables_order = "GPCSE"/' /etc/php/8.3/fpm/php.ini \
    && grep -q '^variables_order = "GPCSE"' /etc/php/8.3/cli/php.ini \
    && grep -q '^variables_order = "GPCSE"' /etc/php/8.3/fpm/php.ini \
    && SYSLOG_USERNAME=probe SYSLOG_PASSWORD=probe php -r 'exit($_ENV["SYSLOG_USERNAME"] === "probe" ? 0 : 1);'

# PimpMyLog 1.7.14 (last commit 2017) calls two functions in inc/global.inc.php
# that were REMOVED in PHP 8.0: get_magic_quotes_gpc() and each(). The page
# therefore fatals on PHP 8.3. Both calls sit inside a single block guarded by
# get_magic_quotes_gpc(); magic quotes do not exist since PHP 5.4, so that block
# is dead code and guarding it with `false` removes both fatals at once.
RUN sed -i 's|if ( get_magic_quotes_gpc() )|if ( false ) /* disabled: magic quotes removed in PHP 8 */|' /var/www/inc/global.inc.php \
    && grep -q 'disabled: magic quotes removed in PHP 8' /var/www/inc/global.inc.php \
    && ! grep -q 'if ( get_magic_quotes_gpc() )' /var/www/inc/global.inc.php \
    && php -l /var/www/inc/global.inc.php \
    && php -l /var/www/index.php

RUN mkdir -p /var/log/net/ /run/php && touch /var/log/net/syslog.log && ln -s /var/log/net/syslog.log /var/www/
RUN chown -R syslog:adm /var/log/net/
RUN chown www-data:www-data /run/php
RUN adduser www-data adm

# run.sh (kept untouched, so its default password is preserved exactly)
# still calls php5; point it at the PHP 8.3 CLI instead of editing that file
RUN ln -s /usr/bin/php /usr/bin/php5

COPY nginx-default /etc/nginx/sites-enabled/default
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf
COPY config.user.php /var/www/
COPY rsyslog.conf /etc/rsyslog.conf
COPY create-user.php /var/www/
COPY run.sh /

#RUN cd /var/www && php -f ./create-user.php && chown www-data:www-data config.auth.user.php

EXPOSE 80 514/udp 514/tcp

CMD ["/run.sh"]
