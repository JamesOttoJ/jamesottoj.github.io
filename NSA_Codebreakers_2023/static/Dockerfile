FROM ubuntu:22.04

ARG DEBIAN_FRONTEND=noninteractive

RUN apt update && apt install -y \
        device-tree-compiler \
        gcc \
        iproute2 \
        iptables \
        iputils-ping \
        libglib2.0-dev \
        libncurses-dev \
        libpixman-1-dev \
        libslirp-dev \
        make \
        netcat \
        ninja-build \
        pkg-config \
        python3 \
        ssh-client \
        wget \
        xz-utils \
    && rm -rf /var/lib/apt/lists/*

COPY qemu-ifup qemu-ifdown /

WORKDIR /build

ARG QEMU_SRC_URL=https://download.qemu.org/qemu-8.0.4.tar.xz
RUN wget -nv ${QEMU_SRC_URL}
RUN QEMU_SRC=$(basename ${QEMU_SRC_URL}) \
    && tar xf ${QEMU_SRC} \
    && cd ${QEMU_SRC%.tar.xz} \
    && ./configure --target-list=aarch64-softmmu,aarch64-linux-user --enable-strip \
    && make -j $(nproc)  \
    && make install \
    && cd / \
    && rm -rf /build

WORKDIR /tmp
CMD /bin/bash
