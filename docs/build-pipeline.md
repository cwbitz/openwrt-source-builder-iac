# Build pipeline

What `roles/buildroot` does for one variant — the pipeline, the source tree and
feeds, the configuration, the build container, the overlay, artifact handling and
the build metadata.

`build.yml` targets `hosts: builders` and loops each machine's `build_variants`
(the current entry is the `build_variant` var); the role slices below run once
per variant and are individually selectable with `--tags`
(`preflight|modules|packages|setup|config|overlay|build|fetch|record`).

## Pipeline

1. **preflight** — asserts the variant carries a usable `source` block and a real
   `source.profile` (no example placeholder), then asserts the Docker CLI/daemon
   and the `docker` Python SDK on the build host, the free-space floor
   (`source_min_free_gb`), and derives the effective job count.
2. **modules** — resolves the active modules: a module mapping with any field set
   counts as enabled (an empty mapping is off), plus each boolean module flag
   (`disable-ipv6`).
3. **packages** — merges the statistics module's package set (removing any
   duplicates from the operator list), deduplicates, and fails on add/remove
   conflicts.
4. **setup** — ensures the pinned build image, clones/fetches the source tree into
   its cache directory, checks out the resolved commit, and updates/installs the
   feeds (see below).
5. **config** — renders the `.config` fragment the project owns (target, device,
   kernel options, kconfig switches, packages/removals, download folder, ccache).
6. **overlay** — renders the enabled modules' uci-defaults, the `network` block
   and the variant's `custom_overlay`, and installs them as `<src>/files/`.
7. **build** — runs the pinned build container (`defconfig` → `download` →
   `world`), then asserts the requested kernel options and kconfig switches
   survived, the kernel config carries them, `.BTF` exists when requested, every
   requested package is still `=y`, and removed packages are absent from
   `.manifest`.
8. **fetch** — copies `bin/targets/...` to the control node; the scratch work
   directory is removed when the role exits.
9. **record** — writes `artifacts/<artifact_id>/build-info/{source.json,config}`.

The runner streams Ansible's output live instead of capturing it until exit, so
task progress and failures are visible while a build runs.

## Source tree and feeds (setup)

The cached tree is `~/.cache/openwrt-builder/source/<key>` with
`key = <target>-<ref>-<sha1(kernel options)[0:8]>`. `setup`:

- copies `roles/buildroot/files/Dockerfile` into
  `~/.cache/openwrt-builder/image/` and builds/keeps the image from it (rebuilt
  only when the Dockerfile changes; never pulled from a registry);
- `git init`s the tree if needed, points `origin` at `source.repo_url`, then
  fetches `source.ref` with `--depth 1`, falling back to a full ref fetch and then
  a whole-history fetch when the server will not serve the requested object;
- resolves `FETCH_HEAD` and checks out that commit **detached**, so the tree is
  always exactly the pinned commit (the resolved commit is recorded);
- runs `./scripts/feeds update -a` for a new tree, an explicitly requested
  `source.update`, or a changed commit, then `./scripts/feeds install -a` and
  records every feed's `HEAD`.

`dl/`, `ccache/` and the Dockerfile context are shared across variants; the
source tree is not shared across kernel configurations.

## Configuration (config)

`templates/config.j2` writes only the settings this project owns, leaving the rest
to the target's own defaults:

- `CONFIG_TARGET_<target>=y`, `CONFIG_TARGET_<target>_<subtarget>=y` and
  `CONFIG_TARGET_<target>_<subtarget>_DEVICE_<profile>=y`;
- `CONFIG_DOWNLOAD_FOLDER="/build/dl"` and `CONFIG_CCACHE=y` (unless disabled);
- the variant's `source.kernel` and `source.config` lines;
- one `CONFIG_PACKAGE_<name>=y` per package and
  `# CONFIG_PACKAGE_<name> is not set` for a `-name` removal entry.

A source tree has no `PACKAGES="-pkg"` syntax, so a removal is the explicit
negative above; the build asserts the package is absent from the generated
`.manifest` afterwards. `source.profile` is resolved by the device line, not by a
`PROFILE=` variable in `.config`.

## Overlay (overlay)

Modules ship first-boot config as uci-defaults under the role:

- `files/overlay/<module>/...` — copied verbatim;
- `templates/overlay/<module>/...` — rendered as Jinja templates by Ansible's
  `template` module.

`network` is not a module: when a variant's `network` block is non-empty, the
role adds the `network` overlay to the selected set and renders
`templates/overlay/network/etc/uci-defaults/89-network`.

The role selects the files of the active modules, asserts no two active modules
target the same path, installs every `etc/uci-defaults/*` and `etc/hotplug.d/*`
file `0755` and every other file `0644`, then copies the variant's
`custom_overlay` directory (if set) over the same tree, preserving modes.
`disabled_services` renders one extra `99-disabled-services` script: a source
build has no `DISABLED_SERVICES` make variable, so the init scripts are disabled
at first boot instead. The assembled tree replaces `<src>/files/` before the
build.

## Container orchestration (build)

`community.docker.docker_container` runs the pinned build image once per variant:

- mounts: the source tree → `/build`, the shared `dl/` → `/build/dl`, the shared
  `ccache/` → `/ccache`, and the variant's work directory → `/work`;
- user: the invoking host uid/gid, so the multi-GB tree and the artifacts stay
  owned by the connection user (no `chown -R` pass);
- env: `OPENWRT_JOBS`, `OPENWRT_PROFILE`, `OPENWRT_PACKAGES`,
  `OPENWRT_EXTRA_FIRMWARE_NAME`, `OPENWRT_ROOTFS_PARTSIZE`;
- `ulimits: nofile:65536:65536` — a bounded `nofile` limit. The host tools use
  fakeroot, whose `faked` daemon closes every fd up to `RLIMIT_NOFILE` on startup;
  the daemon's near-infinite inherited limit turns that into a hang. Keep it
  bounded.
- `cleanup: true`, `detach: false`, `output_logs: true`; a non-zero exit status is
  asserted as a build failure. The container carries `openwrt-builder` labels
  (project, backend, build id, run id).

`templates/build.sh.j2` runs inside the container:

1. optional `make clean` (`source.clean`; keeps `dl/` and `ccache/`);
2. `make defconfig` with `TMPDIR` on disk and `CCACHE_DIR=/ccache`;
3. **asserts every requested kernel option, kconfig switch and package survived
   `defconfig`** — the config is re-derived from the dependency graph, so a
   missing option fails here instead of silently shipping a kernel without
   BTF/ftrace, and a package whose definition is absent from the scanned tree (a
   feed that was not installed) fails instead of shipping an image without it;
   the final config is copied to `/work/config.final`;
4. `make download`, then `make world` (toolchain, kernel, packages, and the
   images — `world` installs them, `make image` no longer exists on these refs;
   `world` is retried once because a parallel host-tool build can start before
   the host tool it links against is installed),
   passing only the `EXTRA_IMAGE_NAME` prefix; the rootfs overlay comes from
   `<tree>/files`, which `include/image.mk` reads directly, and the rootfs size
   comes from `CONFIG_TARGET_ROOTFS_PARTSIZE` in `.config` (`FILES=` would be read
   as the kernel-module list, `ROOTFS_PARTSIZE=` would override the byte value
   `image.mk` derives);
5. verifies the kernel configuration carries each requested option, that
   `vmlinux` has a `.BTF` section when BTF was requested, and that every removed
   package is absent from the target's `.manifest`;
6. lists the produced artifacts.

An interrupted or failed `make build` cleans up after itself: the wrapper
(`scripts/run-build.sh`) runs `playbooks/cleanup_build.yml`, force-removes the
run's containers with their anonymous volumes (`docker rm -f -v`), and deletes
the builder-side work scratch for every variant.

## Kernel and module ABI

A kmod package depends on the exact kernel build it was compiled against:
`include/kernel.mk` adds
`kernel (= <LINUX_VERSION>~<LINUX_VERMAGIC>-r<LINUX_RELEASE>)` to every
`KernelPackage`, and `include/kernel-defaults.mk` derives `LINUX_VERMAGIC` as the
MD5 of every `CONFIG_*=y|m` line in the kernel's effective configuration. The
official feeds are published per hash
(`targets/<target>/kmods/<version>-<release>-<hash>/`), so the hash is what
decides whether a module from "outside" can be installed.

That hash cannot match the release build for a source build: the release is built
with the target's full module set, and every built module contributes its own
`=m` line, so even a variant that leaves `source.kernel` empty has a different
module set and therefore a different hash. The consequence:

- the kmods in a source-built image are only usable on images from the same build;
- `apk add kmod-…` against the official repository fails with
  `unable to select packages: kernel-…: breaks: kmod-…[kernel=…]`;
- to add a module to a source build, list it in the variant's `packages` so it is
  compiled into the image (or serve the build's own `bin/targets/.../packages`);
- the ImageBuilder variant is the reverse trade-off: it runs the official kernel,
  so official kmods install unchanged.

Plain user-space packages are unaffected — the package architecture
(`aarch64_cortex-a53` and so on) still matches the release.

## Artifacts

| Path | Where |
|------|-------|
| `~/.cache/openwrt-builder/work/<machine>-<variant>` | builder-side scratch (build script, rendered overlay), removed after the build |
| `<repo>/artifacts/<artifact_id>/targets/<target>/<subtarget>/` | control-node copy (gitignored) |
| `<repo>/artifacts/<artifact_id>/build-info/` | resolved commit, feeds, final `.config`, `source.json` |

`<artifact_id>` is `<machine>-<variant>-sourcebuilder-<target>-<ref>-<digest>`, where the
digest covers the variant's kernel/config options, packages, profile, rootfs size
and image name — a changed variant therefore gets its own directory instead of
overwriting the previous build's artifacts.

Only `bin/targets` is fetched (never the `bin/packages` tree), with the
`targets/<target>/<subtarget>/` layout preserved: firmware images,
`profiles.json`, `sha256sums`, `.manifest`, and so on. The image file names carry
the version of the checked-out tree (`openwrt-<version>-<variant>-<target>-...`,
for example `openwrt-25.12.6-ap-mediatek-filogic-glinet_gl-mt6000-squashfs-sysupgrade.bin`):
the role enables `CONFIG_IMAGEOPT` plus `CONFIG_VERSIONOPT`/`CONFIG_VERSION_FILENAMES`
(the version prompts are nested under `IMAGEOPT`), so the naming matches the release
and ImageBuilder artifacts instead of dropping the version.
`source.json` records the
build image, repository, ref, resolved commit, target, profile, job count, kernel
options, requested packages, feed commits and a SHA256 of the final config;
`config` is the final `.config` itself. The builder-side `bin/` and work
directories are removed before the next build, so repeated builds do not
accumulate files and re-running the same inputs converges on the same output.
