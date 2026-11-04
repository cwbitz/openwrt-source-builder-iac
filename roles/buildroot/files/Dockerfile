# Pinned build environment for an OpenWrt from-source build.
#
# The host only needs Docker: every buildroot dependency lives here, so the build
# machine stays free of compilers and two IaC projects can share it without one
# project's apt state changing the other's build output.
#
# The tag this image is built and referenced by lives in
# roles/buildroot/vars/main.yml (`source_build_image`); a tag bump is a
# build-environment change and belongs there, not in this file.
# Debian (glibc) on purpose: OpenWrt builds its host tools against glibc, while a
# musl host (Alpine) needs argp/fts/obstack/libintl shims and is not supported.
FROM debian:13-slim

ENV DEBIAN_FRONTEND=noninteractive \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8

# The host toolchain OpenWrt needs: the documented prerequisites (binutils, gcc,
# g++, make, perl, python3, rsync, subversion, unzip, ...) plus every host tool
# the build compiles for itself (cmake, ninja, pkg-config, bison/flex, gettext,
# swig) and the development libraries those builds link against. OpenWrt builds
# the same host tools for every target, so this set is target-independent: one
# image serves every target/profile a variant can select. Package-specific host
# tools (a JDK for Java packages, for example) are out of scope by design: build
# a derived image and point `source_build_image` at it.
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      bash bc binutils bison build-essential bzip2 ca-certificates ccache \
      clang cmake cpio device-tree-compiler diffutils dwarves file findutils flex \
      g++ gawk gcc gcc-multilib g++-multilib gettext git gzip kmod \
      libbz2-dev libelf-dev libdw-dev liblzma-dev libncurses-dev libssl-dev \
      make ninja-build patch perl pkg-config python3 python3-dev \
      python3-setuptools rsync \
      subversion swig tar time unzip wget xz-utils zip zlib1g-dev zstd \
 && rm -rf /var/lib/apt/lists/*

# Build output must stay writable by the invoking host user: the container runs
# as that uid:gid (see roles/buildroot/tasks/build.yml), so no chown pass is
# needed over a multi-GB tree after the build.
ENV HOME=/tmp
WORKDIR /build

CMD ["/bin/bash"]
