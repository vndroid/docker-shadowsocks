FROM alpine:3.19 AS builder

ARG VERSION=3.3.6

WORKDIR /usr/local/src

RUN set -x \
    && apk add --no-cache git \
        build-base \
        cmake \
        c-ares-dev \
        libcap \
        libev-dev \
        libsodium-dev \
        linux-headers \
        mbedtls-dev \
        pcre2-dev \
        python3 \
        bash \
    && git clone -b v${VERSION} https://github.com/shadowsocks/shadowsocks-libev.git . \
    && git submodule update --init --recursive \
    && mkdir -p build && cd build \
    && cmake .. -DCMAKE_INSTALL_PREFIX=/usr/local -DWITH_STATIC=OFF -DCMAKE_BUILD_TYPE=Release \
    && make -j$(getconf _NPROCESSORS_ONLN) \
    && ctest --output-on-failure \
    && python3 ../tests/stress_test.py --bin ../build/shared/bin/ --size 10 \
    && make install

WORKDIR /usr/local/bin
RUN set -x \
    && ls /usr/local/bin/ss-* | xargs -n1 setcap cap_net_bind_service+ep \
    && strip $(scanelf --nobanner -E 'ET_DYN' /usr/local/bin/ss-* | awk '{print $2}')


FROM alpine:3.19

LABEL maintainer="WavE Docker Maintainers <waveworkshop@outlook.com>"

ENV SERVER_ADDR=0.0.0.0
ENV SERVER_PORT=8388
ENV METHOD=aes-256-gcm
ENV TIMEOUT=300
ENV DNS_ADDRS="8.8.8.8,8.8.4.4"
ENV TZ=UTC
ENV ARGS=

COPY --from=builder /usr/local/bin/ss-* /usr/local/bin/
COPY docker-entrypoint.sh /usr/local/bin/

RUN set -x \
    && apk add --no-cache \
        ca-certificates \
        iproute2-ss \
        $(scanelf --needed --nobanner /usr/local/bin/ss-* \
        | awk '{ gsub(/,/, "\nso:", $2); print "so:" $2 }' \
        | sort -u)

ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 8388

STOPSIGNAL SIGINT

CMD ["ss-server"]