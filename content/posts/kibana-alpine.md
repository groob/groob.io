+++
date = "2015-05-30T22:07:08-04:00"
title = "A tiny Kibana Docker container."

+++

I've been using docker-alpine as a base image for a lot of my Docker containers. It has a lot of nice features, the main one being how tiny my containers are. For example, I built an nginx image that is about 15mb in size. For comparision, the official nginx image is 138mb.
Yesterday I set out to build a Kibana dashboard, and ran into an issue: Kibana vendors NodeJS and npm, and the versions it ships are both dependent on glibc. Alpine Linux ships with musl libc library instead, making the glibc dependent binaries [incompatible](https://github.com/gliderlabs/docker-alpine/blob/master/docs/caveats.md#incompatible-binaries).

The solution to running Kibana in a docker-alpine image turned out quite simple - remove the vendored nodejs and install my own, then symlink it to the location that kibana expects it. 

```
FROM alpine:3.1

ENV KIBANA_VERSION 4.0.2

RUN apk add --update nodejs curl && \
    curl -LO https://download.elastic.co/kibana/kibana/kibana-${KIBANA_VERSION}-linux-x64.tar.gz && \
    tar xzf /kibana-${KIBANA_VERSION}-linux-x64.tar.gz -C / && \
    rm /kibana-${KIBANA_VERSION}-linux-x64/node/bin/node && \
    rm /kibana-${KIBANA_VERSION}-linux-x64/node/bin/npm && \
    ln -s /usr/bin/node /kibana-${KIBANA_VERSION}-linux-x64/node/bin/node && \
    ln -s /usr/bin/npm /kibana-${KIBANA_VERSION}-linux-x64/node/bin/npm && \
    sed -i '/elasticsearch_url/s/localhost/elasticsearch/' /kibana-${KIBANA_VERSION}-linux-x64/config/kibana.yml && \
    rm -rf /var/cache/apk/* /kibana-${KIBANA_VERSION}-linux-x64.tar.gz

EXPOSE 5601

CMD ["/kibana-4.0.2-linux-x64/bin/kibana"]
```

The resulting container is about 60mb in size.

  
