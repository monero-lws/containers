ARG MONERO_BRANCH=v0.18.5.1
ARG MONERO_COMMIT_HASH=4f92268d7c16741cfb41e5bbe2aa46cc260a9ea5

FROM stagex/pallet-clang-cmake-busybox AS build
COPY --from=stagex/core-openssl . /
COPY --from=stagex/core-boost . /
COPY --from=stagex/user-libsodium . /
COPY --from=stagex/core-expat . /
COPY --from=stagex/user-zeromq /usr/lib/. /usr/lib
COPY --from=stagex/user-zeromq /usr/include/. /usr/include
COPY --from=stagex/user-eudev . /
COPY --from=stagex/user-libusb . /
COPY --from=stagex/core-sqlite3 . /
COPY --from=stagex/core-make . /
COPY --from=stagex/core-curl . /
COPY --from=stagex/core-ca-certificates . /
COPY --from=stagex/core-git . /
COPY --from=stagex/user-gpg . /
COPY --from=stagex/user-libgcrypt . /
COPY --from=stagex/user-libgpg-error . /
COPY --from=stagex/user-libassuan . /
COPY --from=stagex/user-npth . /

# Build libunbound (not in stagex repo)
WORKDIR /tmp
RUN <<-EOF
        curl -L -O https://www.nlnetlabs.nl/downloads/unbound/unbound-1.24.2.tar.gz
        echo "44e7b53e008a6dcaec03032769a212b46ab5c23c105284aa05a4f3af78e59cdb unbound-1.24.2.tar.gz" | sha256sum -c
        tar -xzf unbound-1.24.2.tar.gz
        cd unbound-1.24.2
        ./configure \
                --disable-shared \
                --enable-static \
                --without-pyunbound \
                --with-libexpat=/usr \
                --with-ssl=/usr \
                --with-libevent=no \
                --without-pythonmodule \
                --disable-flto \
                --with-pthreads \
                --with-libunbound-only \
                --with-pic
        make -j$(nproc)
        make install
EOF

# Switch to Monero source directory
WORKDIR /monero
ARG MONERO_BRANCH
ARG MONERO_COMMIT_HASH
ARG CMAKE_GENERATOR=Ninja

RUN <<-EOF
        git clone \
                --recursive --branch ${MONERO_BRANCH} \
                --depth 1 --shallow-submodules \
                https://github.com/monero-project/monero .
        test `git rev-parse HEAD` = ${MONERO_COMMIT_HASH} || exit 1
        mkdir -p build/release && cd build/release
        cmake -D STATIC=ON -D CMAKE_BUILD_TYPE=Release -D STACK_TRACE=OFF /monero
        cmake --build . -t daemon -j$(nproc)
	strip --strip-unneeded /monero/build/release/bin/monerod
EOF

# git pull and validate ban list
RUN <<-EOF
        git clone https://github.com/Boog900/monero-ban-list
        cd monero-ban-list
        curl -L -O https://raw.githubusercontent.com/Cuprate/cuprate/7b8756fa80e386fb04173d8220c15c86bf9f9888/misc/gpg_keys/boog900.asc
        curl -L -o rucknium.asc https://rucknium.me/pgp.txt
        curl -L -O https://raw.githubusercontent.com/monero-project/monero/004ead1a14d60ff757880c5b16b894b526427829/utils/gpg_keys/jeffro256.asc
        gpg --import boog900.asc
        gpg --import rucknium.asc
        gpg --import jeffro256.asc
        gpg --verify --status-fd 1 ./sigs/boog900.sig ban_list.txt 2>/dev/null
        gpg --verify --status-fd 1 ./sigs/Rucknium.sig ban_list.txt 2>/dev/null
        gpg --verify --status-fd 1 ./sigs/jeffro256.sig ban_list.txt 2>/dev/null
EOF

# Remove static libs and try to keep symlinks
FROM stagex/pallet-clang-cmake-busybox AS staging
COPY --from=stagex/core-musl . /musl
COPY --from=stagex/core-libcxx . /libcxx
COPY --from=stagex/core-libcxxabi . /libcxxabi
COPY --from=stagex/core-libunwind . /libunwind
RUN <<-EOF
        mkdir /staged
        cp -a /musl/usr/lib/*.so* /staged/
        cp -a /libcxx/usr/lib/*.so.1 /staged/
        cp -a /libcxxabi/usr/lib/*.so* /staged/
        cp -a /libunwind/usr/lib/*.so* /staged/
        strip --strip-debug /staged/*
        cp -a /libcxx/usr/lib/libc++.so /staged/
EOF

# Change default username to monero and copy binaries
FROM stagex/core-busybox
COPY --from=staging /staged /usr/lib/
USER root
RUN <<-EOF
        deluser user
	rm -fr /home/user
	addgroup -g 1000 monero
	adduser -g 1000 -D -h /home/monero -G monero -s /bin/sh monero
EOF
USER monero
ENV USER=monero
ENV HOME=/home/monero

# Switch to home directory and install newly built monerod binary
WORKDIR /home/monero
COPY --from=build /monero/build/release/bin/monerod /usr/local/bin/monerod
COPY --from=build /monero/monero-ban-list/ban_list.txt ./ban_list.txt

EXPOSE 18080
EXPOSE 18083
EXPOSE 18089

HEALTHCHECK --interval=30s --timeout=5s CMD wget --spider http://127.0.0.1:18081/get_height || exit 1
ENTRYPOINT [ "/usr/local/bin/monerod", "--non-interactive" ]
CMD ["--rpc-restricted-bind-ip=0.0.0.0", "--rpc-restricted-bind-port=18089", "--no-igd", "--zmq-rpc-bind-ip=0.0.0.0", "--confirm-external-bind", "--zmq-pub=tcp://0.0.0.0:18083", "--enable-dns-blocklist", "--ban-list=/home/monero/ban_list.txt"]
