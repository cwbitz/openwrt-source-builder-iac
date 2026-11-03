# Pinned build environment for an OpenWrt from-source build.
#
# The host only needs Docker: every buildroot dependency lives here, so the build
# machine stays free of compilers and two IaC projects can share it without one
# project's apt state changing the other's build output.
#
# Build:
#   docker build -t openwrt-buildroot:debian13 roles/buildroot/files
#
# A tag bump is a build-environment change: record it in the variant's
# `source.build_image` so the artifacts say which environment produced them.
FROM debian:13-slim

ENV DEBIAN_FRONTEND=noninteractive \
    LANG=C.UTF-8 \
    LC_ALL=C.UTF-8

# The OpenWrt prerequisite toolchain (superset of `make prereq`), plus the
# pieces needed for a BTF/ftrace kernel: elfutils/dwarves (pahole), zlib, zstd.
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      bash bc binutils bison build-essential bzip2 ca-certificates ccache \
      clang cpio device-tree-compiler diffutils dwarves file findutils flex \
      g++ gawk gcc gcc-multilib g++-multilib gettext git gzip kmod \
      libelf-dev libdw-dev liblzma-dev libncurses-dev libssl-dev \
      make patch perl python3 python3-setuptools rsync subversion swig tar \
      time unzip wget xz-utils zip zlib1g-dev zstd \
 && rm -rf /var/lib/apt/lists/*

# Build output must stay writable by the invoking host user: the container runs
# as that uid:gid (see roles/buildroot/tasks/build.yml), so no chown pass is
# needed over a multi-GB tree after the build.
ENV HOME=/tmp
WORKDIR /build

CMD ["/bin/bash"]
