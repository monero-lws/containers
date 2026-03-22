FROM stagex/pallet-clang-gnu-busybox AS build
COPY --from=stagex/core-openssl . /
COPY --from=stagex/user-libevent . /
COPY --from=stagex/core-curl . /
COPY --from=stagex/core-ca-certificates . /
RUN <<-EOF
       curl -O -L https://dist.torproject.org/tor-0.4.9.5.tar.gz
       echo "c949c2f86b348e64891976f6b1e49c177655b23df97193049bf1b8cd3099e179 tor-0.4.9.5.tar.gz" | sha256sum -c
       tar -xf tor-0.4.9.5.tar.gz
       cd tor-0.4.9.5
       CFLAGS="-O2" ./configure --prefix=/usr --sysconfdir=/etc
       make -j$(nproc)
       make install
       mkdir -p /var/lib/tor
EOF

FROM stagex/core-filesystem AS package
COPY --from=stagex/core-musl . /
COPY --from=stagex/core-zlib /usr/lib/*.so* /usr/lib/
COPY --from=stagex/core-libzstd /usr/lib/*.so* /usr/lib/
COPY --from=stagex/core-xz /usr/lib/*.so* /usr/lib/
COPY --from=stagex/core-openssl /usr/lib/*.so* /usr/lib/
COPY --from=stagex/user-libevent /usr/lib/*.so* /usr/lib/
COPY --from=build /usr/bin/tor /usr/bin/
COPY --chown=user:user --from=build /var/lib/tor/ /var/lib/tor/
USER user
ENTRYPOINT ["/usr/bin/tor"]
