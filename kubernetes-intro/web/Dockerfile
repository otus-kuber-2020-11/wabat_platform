FROM alpine:3.11

RUN apk add shadow lighttpd gamin openrc  \
    && usermod -u 1001 lighttpd && groupmod -g 1001 lighttpd \
    && mkdir -p /var/www/localhost/app /var/log/lighttpd /var/lib/lighttpd \
    && sed -i -r 's#\#.*server.port.*=.*#server.port          = 8000#g' /etc/lighttpd/lighttpd.conf \
    && sed -i -r 's#\#.*server.event-handler = "linux-sysepoll".*#server.event-handler = "linux-sysepoll"#g' /etc/lighttpd/lighttpd.conf \
    && sed -i -r 's#.*server.document-root.*=.*#server.document-root = var.basedir + "/app"#g' /etc/lighttpd/lighttpd.conf \
    && mkdir -p /var/lib/lighttpd \
    && chown -R lighttpd:lighttpd /var/www/localhost/ \
    && chown -R lighttpd:lighttpd /var/lib/lighttpd \
    && chown -R lighttpd:lighttpd /var/log/lighttpd \
    && rc-update add lighttpd default \
    && echo "it not works" > /var/www/localhost/app/index.html \
    && chown lighttpd:lighttpd /var/www/localhost/app/index.html

# USER 1001
EXPOSE 8000
ENTRYPOINT ["/usr/sbin/lighttpd", "-D", "-f", "/etc/lighttpd/lighttpd.conf"]
