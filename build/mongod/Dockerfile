FROM alpine:3.7
RUN apk add --no-cache mongodb && \
    rm /usr/bin/mongo /usr/bin/mongos /usr/bin/mongoperf && \
    install -d -o mongodb -g mongodb -m 0755 /srv/mongodb
USER mongodb
CMD ["/usr/bin/mongod", "--config", "/etc/mongod.conf"]