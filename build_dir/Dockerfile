FROM --platform=$TARGETPLATFORM debian:stable-slim AS builder

RUN sed -i 's/security.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list && \
    sed -i 's/deb.debian.org/mirrors.ustc.edu.cn/g' /etc/apt/sources.list && \
    apt-get update && \
    apt-get install build-essential git ca-certificates wget -y --no-install-recommends
RUN wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key | apt-key add - && \
    echo 'deb http://apt.llvm.org/buster/ llvm-toolchain-buster-12 main' >> /etc/apt/sources.list && \
    apt-get update && \
    apt-get install -y clang-12 lld-12

# http://www.acme.com/software/thttpd/
RUN wget http://www.acme.com/software/thttpd/thttpd-2.29.tar.gz && \
    tar -zxvf thttpd-2.29.tar.gz && \
    cd thttpd-2.29 && \
    ./configure && \
    make

RUN git clone --depth=1 https://mirrors.tuna.tsinghua.edu.cn/git/linux-stable.git && \
    cd linux-stable && \
    make headers_install INSTALL_HDR_PATH="/build_glibc/header"

RUN wget https://mirrors.tuna.tsinghua.edu.cn/gnu/glibc/glibc-2.33.tar.xz && \
    tar -Jxf glibc-2.33.tar.xz && \
    mkdir build_glibc && \
    cd build_glibc && \
    CC=clang-12 LD=ld.lld-12 /glibc-2.33/configure --enable-kernel=4.4 --without-selinux --prefix=/usr --with-headers=./header --host=aarch64-linux-gnu && \
    make -j$(nproc) && \
    mkdir /container && \
    make install DESTDIR=/container

RUN wget https://ftp.cn.debian.org/debian/pool/main/g/gcc-10/libstdc++6_10.2.1-6_arm64.deb && \
    mkdir libstdc++ && \
    dpkg-deb -R libstdc++6_10.2.1-6_arm64.deb libstdc++ && \
    cp -r libstdc++/usr/* /container/usr/

FROM scratch

#LABEL maintainer="eric <ericwang2006@gmail.com>"

COPY --from=builder /container/* /

ADD https://github.com/multiarch/qemu-user-static/releases/latest/download/qemu-arm-static /qemu-arm-static
ADD https://raw.githubusercontent.com/ericwang2006/docker_ttnode/master/aarch64/ttnode /ttnode

ENV TZ=Asia/Shanghai
ENV LANG=C.UTF-8

VOLUME ["/config"]

ENTRYPOINT ["/qemu-aarch64-static"]
CMD ["/ttnode","-p","/mnts"]
