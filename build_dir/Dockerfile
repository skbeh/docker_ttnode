FROM debian:bullseye-slim AS builder

RUN sed -i 's/security.debian.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apt/sources.list && \
    sed -i 's/deb.debian.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apt/sources.list && \
    dpkg --add-architecture arm64 && \
    apt-get update && \
    apt-get install build-essential git ca-certificates wget gnupg2 rsync gawk bison python3 ninja-build pkg-config libglib2.0-dev crossbuild-essential-arm64 -y --no-install-recommends && \
    rm -rf /var/lib/{apt,dpkg,cache,log}

#ADD https://github.com/kdrag0n/proton-clang/archive/20210123.zip /proton-clang
RUN git clone --depth=1 https://gitee.com/bfcue/proton-clang.git && \
    rm -rf /proton-clang/.git
ENV PATH="/proton-clang/bin:/usr/local/bin:/usr/bin:/bin"

# http://www.acme.com/software/thttpd/
#RUN wget http://www.acme.com/software/thttpd/thttpd-2.29.tar.gz && \
#    tar -zxvf thttpd-2.29.tar.gz && \
#    cd thttpd-2.29 && \
#    ./configure && \
#    make

RUN ln -sf /usr/bin/cc /proton-clang/bin/clang && \
    ln -sf /usr/bin/c++ /proton-clang/bin/clang++

RUN wget http://gondor.apana.org.au/~herbert/dash/files/dash-0.5.11.3.tar.gz && \
    tar -zxf dash-0.5.11.3.tar.gz && rm -f dash-0.5.11.3.tar.gz && \
    cd dash-0.5.11.3 && \
    export CC="clang" && \
    export CFLAGS="-Os -static" && \
    ./configure --with-libedit && \
    make -j$(nproc) && \
    mkdir -p /container/bin && \
    cp src/dash /container/bin/sh && \
    rm -rf /dash-0.5.11.3

RUN git clone --depth=1 https://mirrors.tuna.tsinghua.edu.cn/git/linux-stable.git && \
    cd linux-stable && \
    rm -rf .git && \
    mkdir /build_glibc && \
    make headers_install ARCH=arm64 INSTALL_HDR_PATH=/build_glibc && \
    rm -rf /linux-stable

RUN wget -q https://mirrors.tuna.tsinghua.edu.cn/gnu/glibc/glibc-2.33.tar.xz && \
    tar -Jxf glibc-2.33.tar.xz && \
    rm -f glibc-2.33.tar.xz && \
    cd build_glibc && \
    export CFLAGS='-O3 -fgraphite-identity -floop-nest-optimize -fno-semantic-interposition -fipa-pta -fdevirtualize-at-ltrans -fno-common -fno-math-errno -fno-trapping-math -fno-plt -pipe' && \
    /glibc-2.33/configure --enable-kernel=4.4 --without-selinux --prefix=/usr --with-headers=/build_glibc/include --host=aarch64-linux-gnu --disable-werror && \
    make -j$(nproc) && \
    make install DESTDIR=/container && \
    rm -rf /build_glibc

RUN wget -q https://mirrors.tuna.tsinghua.edu.cn/debian/pool/main/g/gcc-10/libstdc++6_10.2.1-6_arm64.deb && \
    mkdir libstdc++ && \
    dpkg-deb -R libstdc++6_10.2.1-6_arm64.deb libstdc++ && \
    cp libstdc++/usr/lib/aarch64-linux-gnu/* /container/usr/lib64/ && \
    wget -q https://mirrors.tuna.tsinghua.edu.cn/debian/pool/main/g/gcc-10/libgcc-s1_10.2.1-6_arm64.deb && \
    mkdir libgcc && \
    dpkg-deb -R libgcc-s1_10.2.1-6_arm64.deb libgcc && \
    cp libgcc/lib/aarch64-linux-gnu/* /container/lib64/ && \
    rm -rf *.deb libstdc++ libgcc

RUN git clone --depth=1 --single-branch -j$(nproc) -b v5.2.0 https://mirrors.tuna.tsinghua.edu.cn/git/qemu.git && \
    mkdir build_qemu && \
    cd build_qemu && \
    export CC=/proton-clang/bin/clang CXX=/proton-clang/bin/clang++ LD=/proton-clang/bin/lld LLVM=1 LLVM_IAS=1 && \
    export CFLAGS='-O3 -fno-semantic-interposition -fno-trapping-math -fno-plt -fno-strict-aliasing -fno-math-errno -fno-trapping-math -fno-common -pipe' && \
    export CXXFLAGS=${CFLAGS} && \
    ../qemu/configure \
        --prefix=/usr \
        --enable-linux-user \
        --disable-debug-info \
        --disable-bsd-user \
        --disable-werror \
        --disable-system \
        --disable-tools \
        --disable-docs \
        --disable-gtk \
        --disable-gnutls \
        --disable-nettle \
        --disable-gcrypt \
        --disable-glusterfs \
        --disable-libnfs \
        --disable-libiscsi \
        --disable-vnc \
        --disable-kvm \
        --disable-libssh \
        --disable-libxml2 \
        --disable-vde \
        --disable-sdl \
        --disable-opengl \
        --disable-xen \
        --disable-fdt \
        --disable-vhost-net \
        --disable-vhost-crypto \
        --disable-vhost-user \
        --disable-vhost-vsock \
        --disable-vhost-scsi \
        --disable-tpm \
        --disable-qom-cast-debug \
        --disable-capstone \
        --disable-zstd \
        --disable-linux-io-uring \
        --target-list=aarch64-linux-user \
        --static && \
    make -j$(nproc) && \
    mkdir qemu_static && \
    make DESTDIR=/build_qemu/qemu_static install && \
    mv qemu_static/usr/bin/qemu-aarch64 /qemu-aarch64-static && \
    rm -rf /build_qemu

ADD https://raw.githubusercontent.com/ericwang2006/docker_ttnode/master/aarch64/ttnode /container/ttnode
RUN chmod +x /container/ttnode

FROM alpine

COPY --from=builder /container/ /
COPY --from=builder /qemu-aarch64-static /
#COPY start.sh /

ENV TZ=Asia/Shanghai LANG=C.UTF-8

VOLUME ["/config"]

CMD /qemu-aarch64-static /ttnode -p /mnts && sleep infinity
#ENTRYPOINT /start.sh
