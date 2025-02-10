FROM golang:alpine AS dataplaneapi-builder

ENV DATAPLANE_MINOR 3.1.0
ENV DATAPLANE_V2_MINOR 2.9.9
ENV DATAPLANE_URL https://github.com/haproxytech/dataplaneapi.git

RUN apk add --no-cache ca-certificates git make && \
    git clone "${DATAPLANE_URL}" "${GOPATH}/src/github.com/haproxytech/dataplaneapi" && \
    cd "${GOPATH}/src/github.com/haproxytech/dataplaneapi" && \
    git checkout "v${DATAPLANE_MINOR}" && \
    make build && cp build/dataplaneapi /dataplaneapi && \
    make clean && \
    git checkout "v${DATAPLANE_V2_MINOR}" && \
    make build && cp build/dataplaneapi /dataplaneapi-v2

FROM debian:bookworm-slim AS openssl-builder

ENV OPENSSL_URL https://github.com/quictls/openssl/archive/refs/tags/openssl-3.3.0-quic1.tar.gz

ENV DEBIAN_FRONTEND noninteractive

RUN apt-get update && \
    apt-get install -y --no-install-recommends build-essential ca-certificates curl && \
    curl -sfSL "${OPENSSL_URL}" -o openssl.tar.gz && \
    mkdir -p /tmp/openssl && \
    tar -xzf openssl.tar.gz -C /tmp/openssl --strip-components=1 && \
    rm -f openssl.tar.gz && \
    cd /tmp/openssl && \
    ./config --libdir=lib --prefix=/opt/quictls && \
    make -j $(nproc) && \
    make install && \
    rm -rf /tmp/openssl

FROM debian:bookworm-slim

MAINTAINER Dinko Korunic <dkorunic@haproxy.com>

LABEL Name HAProxy
LABEL Release Community Edition
LABEL Vendor HAProxy
LABEL Version 2.6.21
LABEL RUN /usr/bin/docker -d IMAGE

ENV HAPROXY_BRANCH 2.6
ENV HAPROXY_MINOR 2.6.21
ENV HAPROXY_SHA256 b905898bf762adbde180a866dbae2e39a4a377767cf3dd08694be9e56e876274
ENV HAPROXY_SRC_URL http://www.haproxy.org/download

ENV HAPROXY_UID haproxy
ENV HAPROXY_GID haproxy

ENV DEBIAN_FRONTEND noninteractive

COPY --from=dataplaneapi-builder /dataplaneapi /usr/local/bin/dataplaneapi
COPY --from=dataplaneapi-builder /dataplaneapi-v2 /usr/local/bin/dataplaneapi-v2
COPY --from=openssl-builder /opt/quictls /opt/quictls

RUN apt-get update && \
    apt-get install -y --no-install-recommends procps zlib1g "libpcre2-*" liblua5.4-0 libatomic1 tar curl socat ca-certificates && \
    apt-get install -y --no-install-recommends gcc make libc6-dev libpcre2-dev zlib1g-dev liblua5.4-dev && \
    curl -sfSL "${HAPROXY_SRC_URL}/${HAPROXY_BRANCH}/src/haproxy-${HAPROXY_MINOR}.tar.gz" -o haproxy.tar.gz && \
    echo "$HAPROXY_SHA256 *haproxy.tar.gz" | sha256sum -c - && \
    groupadd "$HAPROXY_GID" && \
    useradd -g "$HAPROXY_GID" "$HAPROXY_UID" && \
    mkdir -p /tmp/haproxy && \
    tar -xzf haproxy.tar.gz -C /tmp/haproxy --strip-components=1 && \
    rm -f haproxy.tar.gz && \
    make -C /tmp/haproxy -j"$(nproc)" TARGET=linux-glibc CPU=generic USE_PCRE2=1 USE_PCRE2_JIT=1 \
                            USE_TFO=1 USE_LINUX_TPROXY=1 USE_LUA=1 USE_GETADDRINFO=1 \
                            USE_PROMEX=1 USE_SLZ=1 \
                            USE_OPENSSL=1 USE_PTHREAD_EMULATION=1 \
                            SSL_INC=/opt/quictls/include SSL_LIB=/opt/quictls/lib USE_QUIC=1 \
                            LDFLAGS="-L/opt/quictls/lib -Wl,-rpath,/opt/quictls/lib" \
                            all && \
    make -C /tmp/haproxy TARGET=linux-glibc install-bin install-man && \
    ln -s /usr/local/sbin/haproxy /usr/sbin/haproxy && \
    mkdir -p /var/lib/haproxy && \
    chown "$HAPROXY_UID:$HAPROXY_GID" /var/lib/haproxy && \
    mkdir -p /usr/local/etc/haproxy && \
    ln -s /usr/local/etc/haproxy /etc/haproxy && \
    cp -R /tmp/haproxy/examples/errorfiles /usr/local/etc/haproxy/errors && \
    rm -rf /tmp/haproxy && \
    apt-get purge -y --auto-remove gcc make libc6-dev libpcre2-dev zlib1g-dev liblua5.4-dev && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* && \
    chmod +x /usr/local/bin/dataplaneapi && \
    ln -s /usr/local/bin/dataplaneapi /usr/bin/dataplaneapi && \
    chmod +x /usr/local/bin/dataplaneapi-v2 && \
    ln -s /usr/local/bin/dataplaneapi-v2 /usr/bin/dataplaneapi-v2 && \
    touch /usr/local/etc/haproxy/dataplaneapi.yml && \
    chown "$HAPROXY_UID:$HAPROXY_GID" /usr/local/etc/haproxy/dataplaneapi.yml && \
    echo "/opt/quictls/lib" > /etc/ld.so.conf.d/quictls.conf && \
    mkdir -p /opt/quictls/ssl && \
    rm -rf /opt/quictls/ssl/certs && \
    ln -s /etc/ssl/certs /opt/quictls/ssl/certs && \
    ldconfig

COPY haproxy.cfg /usr/local/etc/haproxy
COPY docker-entrypoint.sh /

STOPSIGNAL SIGUSR1

ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["haproxy", "-f", "/usr/local/etc/haproxy/haproxy.cfg"]
