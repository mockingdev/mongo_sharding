FROM alpine:3.7
RUN apk add --no-cache mongodb && \
    rm /usr/bin/mongo /usr/bin/mongod /usr/bin/mongoperf
USER mongodb
CMD ["/usr/bin/mongos", "--config", "/etc/mongos.conf"]