# RISC-V64 cross-compilation environment (Debian Bookworm)
# - Includes riscv64-linux-gnu cross toolchain and common build deps
# - Adds qemu-user to run simple riscv64 binaries via emulation
# - Performs a tiny sanity check compile

FROM debian:bookworm

ARG DEBIAN_FRONTEND=noninteractive

# 1) Enable source repositories and riscv64 multiarch
# 2) Install base tools, cross toolchain, qemu, and useful build deps
RUN set -eux; \
    sed -n 's/^deb /deb-src /p' /etc/apt/sources.list >> /etc/apt/sources.list; \
    dpkg --add-architecture riscv64; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        curl \
        wget \
        xz-utils \
        file \
        git \
        build-essential \
        g++ \
        g++-12 \
        gfortran-12 \
        pkg-config \
        cmake \
        ninja-build \
        ccache \
        python3 \
        python3-venv \
        python3-pip \
        gawk \
        flex \
        bison \
        texinfo \
        libgmp-dev \
        libmpfr-dev \
        libmpc-dev \
        libisl-dev \
        zlib1g-dev \
        libexpat1-dev \
        libzstd-dev \
        libbz2-dev \
        rsync \
        unzip \
        locales \
        dpkg-dev \
        debhelper \
        devscripts \
        quilt \
        autoconf2.69 \
        automake \
        libtool \
        lsb-release \
        gdc-12 \
        gnat-12 \
        time \
    # Cross compile toolchain and essentials (no crossbuild-essential on Debian)
    binutils-riscv64-linux-gnu \
    gcc-riscv64-linux-gnu \
    g++-riscv64-linux-gnu \
    libc6-dev-riscv64-cross \
    libstdc++-12-dev-riscv64-cross \
    linux-libc-dev-riscv64-cross \
    qemu-user \
    qemu-user-static; \
    # Try to pull gcc-12 build dependencies if available in this suite
    apt-get -y build-dep gcc-12 || true; \
    # locale
    sed -i '/^# *en_US.UTF-8 UTF-8/s/^# *//' /etc/locale.gen; \
    locale-gen; \
    rm -rf /var/lib/apt/lists/*

ENV LANG=en_US.UTF-8 \
    LC_ALL=en_US.UTF-8 \
    GDC=/usr/bin/gdc-12 \
    CC=riscv64-linux-gnu-gcc \
    CXX=riscv64-linux-gnu-g++ \
    AR=riscv64-linux-gnu-ar \
    AS=riscv64-linux-gnu-as \
    LD=riscv64-linux-gnu-ld \
    STRIP=riscv64-linux-gnu-strip \
    RANLIB=riscv64-linux-gnu-ranlib \
    OBJDUMP=riscv64-linux-gnu-objdump \
    OBJCOPY=riscv64-linux-gnu-objcopy \
    NM=riscv64-linux-gnu-nm \
    CROSS_COMPILE=riscv64-linux-gnu- \
    DEB_BUILD_OPTIONS="parallel=20"

# Target triplet and sysroot prefix for cross libraries
ENV TARGET=riscv64-linux-gnu \
    TARGET_PREFIX=/usr/riscv64-linux-gnu \
    PKG_CONFIG_PATH=/usr/riscv64-linux-gnu/lib/pkgconfig:/usr/lib/riscv64-linux-gnu/pkgconfig:$PKG_CONFIG_PATH

# Quick sanity check: compile a tiny file to object (no linking needed)
RUN set -eux; \
    echo 'int main(){return 0;}' > /tmp/hello.c; \
    riscv64-linux-gnu-gcc -c /tmp/hello.c -o /tmp/hello.o; \
    file /tmp/hello.o | grep -qi 'RISC-V' || (echo 'Cross-compiler check failed' && exit 1); \
    rm -f /tmp/hello.c /tmp/hello.o

# Build and install riscv64 target libraries from source:
# - zlib-1.3.1 (from zlib.net)
# - isl-0.27 (from libisl)
# - gmp, mpfr, mpc (minimum versions satisfied; choose recent stable)
RUN set -eux; \
    export MAKEFLAGS=-j"$(nproc)"; \
    build="/tmp/build-cross-libs"; mkdir -p "$build"; cd "$build"; \
    # Fetch sources
    wget -q https://zlib.net/zlib-1.3.1.tar.gz; \
    wget -q https://libisl.sourceforge.io/isl-0.27.tar.xz; \
    wget -q https://ftp.gnu.org/gnu/gmp/gmp-6.3.0.tar.xz; \
    wget -q https://www.mpfr.org/mpfr-4.2.1/mpfr-4.2.1.tar.xz; \
    wget -q https://ftp.gnu.org/gnu/mpc/mpc-1.3.1.tar.gz; \
    wget -q https://github.com/facebook/zstd/releases/download/v1.5.6/zstd-1.5.6.tar.gz; \
    \
    build_dir() { mkdir -p "$1-build"; cd "$1-build"; }; \
    host_cfg="--build=$(dpkg-architecture -qDEB_BUILD_GNU_TYPE) --host=${TARGET}"; \
    \
    # 1) GMP (required by MPFR/MPC/ISL). Cross compile; disable assembly to avoid runtime probes.
    tar -xf gmp-6.3.0.tar.xz; \
    build_dir gmp-6.3.0; \
    ../gmp-6.3.0/configure $host_cfg \
        --prefix="${TARGET_PREFIX}" --libdir="${TARGET_PREFIX}/lib" \
        --enable-cxx --enable-shared --disable-static --disable-assembly \
        CC="${TARGET}-gcc" CXX="${TARGET}-g++" AR="${TARGET}-ar" RANLIB="${TARGET}-ranlib"; \
    make; make install; cd ..; \
    
    # 2) MPFR
    tar -xf mpfr-4.2.1.tar.xz; \
    build_dir mpfr-4.2.1; \
    ../mpfr-4.2.1/configure $host_cfg \
        --prefix="${TARGET_PREFIX}" --libdir="${TARGET_PREFIX}/lib" \
        --with-gmp="${TARGET_PREFIX}" --enable-shared --disable-static \
        CC="${TARGET}-gcc"; \
    make; make install; cd ..; \
    
    # 3) MPC
    tar -xf mpc-1.3.1.tar.gz; \
    build_dir mpc-1.3.1; \
    ../mpc-1.3.1/configure $host_cfg \
        --prefix="${TARGET_PREFIX}" --libdir="${TARGET_PREFIX}/lib" \
        --with-gmp="${TARGET_PREFIX}" --with-mpfr="${TARGET_PREFIX}" \
        --enable-shared --disable-static CC="${TARGET}-gcc"; \
    make; make install; cd ..; \
    
    # 4) ISL (use GMP backend)
    tar -xf isl-0.27.tar.xz; \
    build_dir isl-0.27; \
    ../isl-0.27/configure $host_cfg \
        --prefix="${TARGET_PREFIX}" --libdir="${TARGET_PREFIX}/lib" \
        --with-gmp-prefix="${TARGET_PREFIX}" --enable-shared --disable-static \
        CC="${TARGET}-gcc"; \
    make; make install; cd ..; \
    
    # 5) zlib
    tar -xf zlib-1.3.1.tar.gz; \
    cd zlib-1.3.1; \
    CC="${TARGET}-gcc" AR="${TARGET}-ar" RANLIB="${TARGET}-ranlib" \
      ./configure --prefix="${TARGET_PREFIX}" --libdir="${TARGET_PREFIX}/lib"; \
    make; make install; cd ..; \

        # 6) zstd (Zstandard) - build shared library for target
        tar -xf zstd-1.5.6.tar.gz; \
        cd zstd-1.5.6; \
        make -C lib \
            CC="${TARGET}-gcc" AR="${TARGET}-ar" RANLIB="${TARGET}-ranlib" \
            SHARED=1 ZSTD_LEGACY_SUPPORT=0; \
        make -C lib install \
            PREFIX="${TARGET_PREFIX}" LIBDIR="${TARGET_PREFIX}/lib" \
            CC="${TARGET}-gcc" AR="${TARGET}-ar" RANLIB="${TARGET}-ranlib" \
            SHARED=1 ZSTD_LEGACY_SUPPORT=0; \
        cd ..; \
    
    # Cleanup
    rm -rf "$build"

# Sanity link test against target libs (no execution)
RUN set -eux; \
        printf '%s\n' \
            '#include <zlib.h>' \
            '#include <gmp.h>' \
            '#include <mpfr.h>' \
            '#include <mpc.h>' \
        '#include <zstd.h>' \
            '#include <isl/ctx.h>' \
            'int main(void){' \
            '  mpz_t z; mpz_init(z); mpz_clear(z);' \
            '  mpfr_t r; mpfr_init2(r, 53); mpfr_clear(r);' \
            '  mpc_t c; mpc_init2(c, 53); mpc_clear(c);' \
            '  isl_ctx *ctx = isl_ctx_alloc(); isl_ctx_free(ctx);' \
        '  (void)ZSTD_isError(0);' \
            '  const char *v = zlibVersion(); (void)v;' \
            '  return 0;' \
            '}' \
            > /tmp/linkcheck.c; \
        ${TARGET}-gcc /tmp/linkcheck.c -o /tmp/linkcheck \
            -Wl,-rpath-link,${TARGET_PREFIX}/lib \
    -lisl -lmpc -lmpfr -lgmp -lzstd -lz; \
        file /tmp/linkcheck | grep -qi 'ELF 64-bit.*RISC-V' || (echo 'Link check failed' && exit 1); \
        rm -f /tmp/linkcheck.c /tmp/linkcheck

# Working directory for your source code (mount your repo here at runtime)
WORKDIR /work

CMD ["bash"]
