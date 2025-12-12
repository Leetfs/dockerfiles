# 本 Dockerfile 用于编译 openEuler RISC-V 64 位架构的软件包

FROM ubuntu:24.04

RUN apt-get update && apt-get install -y \
    nano \
    build-essential \
    git \
    wget \
    curl \
    gcc \
    g++ \
    make \
    python3 \
    python3-pip \
    autoconf \
    automake \
    libtool \
    pkg-config \
    ccache \
    bc \
    ca-certificates \
    sudo \
    gawk \
    bison \
    flex \
    libncurses5-dev \
    libssl-dev \
    zlib1g-dev \
    libexpat1-dev \
    libcurl4-openssl-dev \
    # QEMU 用户模式模拟器 - 必需用于运行 RISC-V 二进制文件
    qemu-user-static \
    binfmt-support \
    # OBS 构建工具依赖
    rpm \
    rpm2cpio \
    cpio \
    obs-build \
    createrepo-c \
    # OBS 客户端工具
    osc \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# 交叉编译工具链
RUN curl -fL https://github.com/riscv-collab/riscv-gnu-toolchain/releases/download/2025.01.20/riscv64-glibc-ubuntu-22.04-gcc-nightly-2025.01.20-nightly.tar.xz -o /tmp/riscv64-toolchain.tar.xz && \
    mkdir -p /opt/riscv && \
    tar -xf /tmp/riscv64-toolchain.tar.xz -C /opt/riscv --strip-components=1 && \
    rm /tmp/riscv64-toolchain.tar.xz

# 设置交叉编译环境变量
ENV PATH=/opt/riscv/bin:$PATH \
    CC=riscv64-unknown-linux-gnu-gcc \
    CXX=riscv64-unknown-linux-gnu-g++ \
    AR=riscv64-unknown-linux-gnu-ar \
    AS=riscv64-unknown-linux-gnu-as \
    LD=riscv64-unknown-linux-gnu-ld \
    STRIP=riscv64-unknown-linux-gnu-strip \
    RANLIB=riscv64-unknown-linux-gnu-ranlib \
    OBJDUMP=riscv64-unknown-linux-gnu-objdump \
    OBJCOPY=riscv64-unknown-linux-gnu-objcopy \
    NM=riscv64-unknown-linux-gnu-nm \
    CROSS_COMPILE=riscv64-unknown-linux-gnu-

# 配置 QEMU 用户模式,使系统能够自动执行 RISC-V 二进制文件
RUN update-binfmts --enable qemu-riscv64 || true

# 验证交叉编译工具链是否正常工作
RUN echo 'int main(){return 0;}' > /tmp/test.c && \
    riscv64-unknown-linux-gnu-gcc -c /tmp/test.c -o /tmp/test.o && \
    file /tmp/test.o | grep -qi 'RISC-V' && \
    rm -f /tmp/test.c /tmp/test.o

# 验证 QEMU 模拟器是否工作
RUN echo 'int main(){return 0;}' > /tmp/test.c && \
    riscv64-unknown-linux-gnu-gcc /tmp/test.c -o /tmp/test && \
    qemu-riscv64-static /tmp/test && \
    rm -f /tmp/test.c /tmp/test

# 设置工作目录
VOLUME ["/workspace"]
WORKDIR /workspace

CMD ["/bin/bash"]
