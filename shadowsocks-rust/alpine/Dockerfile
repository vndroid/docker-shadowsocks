FROM rust:1.94-alpine3.23 AS builder

ARG TARGETARCH
ARG BUILDARCH

ARG VERSION=1.24.0

RUN set -x \
    && apk add --no-cache build-base git \
    && echo "Building for ${TARGETARCH} on ${BUILDARCH}"

WORKDIR /src

RUN set -x \
    && git clone -b v${VERSION} --single-branch --depth=1 https://github.com/shadowsocks/shadowsocks-rust.git . \
    && apkArch="$(apk --print-arch)" \
    && case "$apkArch" in \
        "x86") \
            RUST_TARGET="i686-unknown-linux-musl" \
            MUSL="i686-linux-musl" \
            SHA512="5047afc68170a2910895db2dfa448227e71a984bfa2130a1bc946fd1015d722b80b15e4abf90c64300815aa84fe781cc8b8a72f10174f9dce96169e035911880" \
        ;; \
        "x86_64") \
            RUST_TARGET="x86_64-unknown-linux-musl" \
            MUSL="x86_64-linux-musl" \
            SHA512="52abd1a56e670952116e35d1a62e048a9b6160471d988e16fa0e1611923dd108a581d2e00874af5eb04e4968b1ba32e0eb449a1f15c3e4d5240ebe09caf5a9f3" \
        ;; \
        "aarch64") \
            RUST_TARGET="aarch64-unknown-linux-musl" \
            MUSL="aarch64-linux-musl" \
            SHA512="8695ff86979cdf30fbbcd33061711f5b1ebc3c48a87822b9ca56cde6d3a22abd4dab30fdcd1789ac27c6febbaeb9e5bde59d79d66552fae53d54cc1377a19272" \
        ;; \
        *) \
            echo "ERROR: Doesn't support $apkArch architecture!" \
            exit 1 \
        ;; \
    esac \
    && wget -qO- "https://github.com/AaronChen0/musl-cc-mirror/releases/download/2021-09-23/$MUSL-cross.tgz" | tee >(sha512sum | grep -q "^$SHA512 " || exit 1) | tar -xzC . \
    && TOOLCHAIN="/src/$MUSL-cross/bin/$MUSL-gcc" \
    && if [ -x "$TOOLCHAIN" ] && "$TOOLCHAIN" --version >/dev/null 2>&1; then \
        CC="$TOOLCHAIN"; \
        echo "INFO: Using downloaded cross toolchain: $CC"; \
    else \
        CC="gcc"; \
        echo "INFO: downloaded cross toolchain is not executable on this builder, fallback to native gcc"; \
    fi \
    && echo "CC=$CC" \
    && rustup target add "$RUST_TARGET" \
    && RUSTFLAGS="-C linker=$CC" CC=$CC cargo build --target "$RUST_TARGET" --release --features "full" \
    && mv target/$RUST_TARGET/release/ss* target/release/ 

FROM alpine:3.23 AS ssserver

LABEL maintainer="WavE Docker Maintainers <waveworkshop@outlook.com>"

COPY --from=builder /src/target/release/sslocal /usr/local/bin/
COPY --from=builder /src/target/release/ssserver /usr/local/bin/
COPY --from=builder /src/target/release/ssmanager /usr/local/bin/
COPY --from=builder /src/target/release/ssservice /usr/local/bin/
COPY --from=builder /src/examples/config.json /etc/shadowsocks-rust/
COPY docker-entrypoint.sh /usr/local/bin/

RUN set -x \
    && apk add --no-cache iproute2-ss

ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 8388

STOPSIGNAL SIGINT

CMD ["ssserver", "--log-without-time", "-a", "nobody", "-c", "/etc/shadowsocks-rust/config.json"]