FROM stagex/pallet-clang-gnu-busybox AS build
COPY --from=stagex/core-openssl . /
COPY --from=stagex/user-libevent . /
COPY --from=stagex/core-curl . /
COPY --from=stagex/core-ca-certificates . /
RUN <<-EOF
       curl -O -L https://dist.torproject.org/tor-0.4.9.11.tar.gz
       echo "2e6c1720118c812acf0079fd47cf91b6bfaba5d766c321c4d3d2a28d6a11a8ed  tor-0.4.9.11.tar.gz" | sha256sum -c
       tar -xf tor-0.4.9.11.tar.gz
       cd tor-0.4.9.11
       CFLAGS="-O2" ./configure --prefix=/usr --sysconfdir=/etc
       make -j$(nproc)
       make install
       strip --strip-debug /usr/bin/tor
EOF
FROM stagex/pallet-clang-gnu-busybox AS staging
COPY --from=stagex/core-musl . /musl
COPY --from=stagex/core-zlib . /zlib
COPY --from=stagex/core-libzstd . /libzstd
COPY --from=stagex/core-xz . /xz
COPY --from=stagex/core-openssl . /openssl
COPY --from=stagex/user-libevent . /libevent
RUN <<-EOF
        mkdir /staged
	cp -a /musl/usr/lib/*.so* /staged/
        cp -a /zlib/usr/lib/*.so* /staged/
        cp -a /libzstd/usr/lib/*.so* /staged/
        cp -a /xz/usr/lib/*.so* /staged/
        cp -a /openssl/usr/lib/*.so* /staged/
        cp -a /libevent/usr/lib/*.so* /staged/
	strip --strip-debug /staged/*.so
EOF
FROM stagex/core-filesystem AS package
COPY --from=staging /staged/. /usr/lib/
COPY --from=build /usr/bin/tor /usr/bin/
USER user
ENTRYPOINT ["/usr/bin/tor"]
