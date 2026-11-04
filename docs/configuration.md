# Configuration

How the inventory is laid out, the per-variant build schema (including the
`source` block), module/package selection, network/system/statistics settings,
build resources, and where secrets live.

## Inventory model

One inventory host = one build **machine**; a machine's **variants** live in its
own `host_vars`.

| Location | Purpose |
|----------|---------|
| `inventory/hosts.yml` | the `builders` group: one host per machine, each with its `ansible_host` |
| `inventory/group_vars/all/main.yml` | committed project-owned baseline: `ssh_identities` + the connection-plugin interface and the per-machine vault path |
| `inventory/group_vars/builders/main.yml` | machine-wide operator overrides, local-only (e.g. `create_admin_account`, `docker_registry_mirrors`) |
| `inventory/host_vars/<machine>/main.yml` | that machine's `build_variants` list and `source_min_free_gb` |
| `inventory/host_vars/<machine>/vault.yml` | that machine's connection secrets |
| `inventory/knob_schema.yml` | committed validation expression per top-level operator knob |

The committed files are `inventory/hosts.yml.example`, the project-owned baseline
`group_vars/all/main.yml`, and the `group_vars/builders/*.example` +
`host_vars/example/` templates; the real files are gitignored.

## Knob validation

`inventory/knob_schema.yml` declares one validation expression per top-level
operator knob, and `playbooks/validate_knobs.yml` asserts every value against it
plus the structure of each `build_variants` entry (including the nested `source`,
`network`, `system`, `statistics` and `administration` blocks). `make lint` and
`make build` run it before any change, so a malformed knob fails with its name and
the received value instead of midway through a build. The schema must cover every
top-level key of `inventory/host_vars/example/main.yml`; the play fails when they
drift apart.

The variant checks also reject unknown keys inside a block, duplicate variant
names, and any remaining example placeholder (`<release-tag-or-commit>`,
`<target>/<subtarget>`, `<profile>`).

## Variant schema

Each entry in a machine's `build_variants` list describes one build. `name` and
the `source` block's `ref`, `target` and `profile` are required; every other key
is optional and falls back to the role default.

| Key | Meaning |
|-----|---------|
| `name` | Variant name; forms the build id `<machine>-<variant>` |
| `source` | **[REQUIRED]** the source build block (see below) |
| `source.profile` | **[REQUIRED]** device profile from `target/linux/<target>/<subtarget>/profiles` (`make info` lists them) |
| `packages` | Exact package list; a `-pkg` entry removes the package from the image (see [Packages](#packages)) |
| `custom_overlay` | Control-node directory copied over the image rootfs after modules (see [Custom overlay](#custom-overlay)) |
| `network` | LAN first-boot configuration (see [Network](#network)) |
| `system` / `statistics` / `administration` | Module mappings; listing any field enables the module (see [Modules](#modules)) |
| `disable-ipv6` | Boolean flag; `true` disables IPv6, RA, and DHCPv6 on LAN/WAN |
| `extra_firmware_name` | Tag inserted into the output filename after the version (keep it distinct per variant) |
| `disabled_services` | Space-separated init.d service names, disabled by the overlay at first boot |
| `rootfs_partsize` | Root filesystem partition size in MB, 64–32768 (empty ⇒ target default) |

No firmware secrets are baked into the image: set the WAN (PPPoE/DHCP) and the
root password after first boot.

## Source block

```yaml
source:
  ref: v25.12.6
  target: mediatek/filogic
  profile: glinet_gl-mt6000
  repo_url: https://git.openwrt.org/openwrt/openwrt.git
  update: false
  jobs: ""
  clean: false
  ccache: true
  kernel:
    - CONFIG_KERNEL_DEBUG_INFO=y
    - CONFIG_KERNEL_DEBUG_INFO_BTF=y
    - CONFIG_KERNEL_DEBUG_INFO_REDUCED=n
```

| Key | Default | Meaning |
|-----|---------|---------|
| `ref` | — | **[REQUIRED]** tag, branch or commit-ish; pin a tag for a reproducible build |
| `target` | — | **[REQUIRED]** `<target>/<subtarget>`, e.g. `mediatek/filogic`, `x86/64` |
| `profile` | — | **[REQUIRED]** device profile of that target, e.g. `glinet_gl-mt6000`, `generic` |
| `repo_url` | `https://git.openwrt.org/openwrt/openwrt.git` | Upstream (or mirror) repository |
| `update` | `false` | Run `feeds update -a` before building; the tree still fetches the ref |
| `jobs` | `""` | Parallel make jobs; empty ⇒ `min(vCPU, RAM-GB / 2)` from the RAM the guest reports — a ballooned VM shrinks it, so pin it on a build host |
| `clean` | `false` | Run `make clean` before the build (keeps `dl/` and `ccache/`) |
| `ccache` | `true` | Share the `ccache/` directory across builds; writes `CONFIG_DEVEL=y` + `CONFIG_CCACHE=y` + `CONFIG_CCACHE_DIR="/ccache"` (the symbol is only visible with the developer options) |
| `kernel` | `[]` | Extra kernel options merged into `.config`; every entry must be `CONFIG_<NAME>=y\|m\|n` (`=n` is asserted as the `# … is not set` line kconfig writes back) |

### Pinning and the cache key

The tree is cached per `(target, ref, kernel options)`:
`~/.cache/openwrt-builder/source/<target>-<ref>-<sha1(kernel options)[0:8]>`. Two
variants that differ only in kernel options therefore get separate trees, because
a different kernel configuration forces a full kernel + kmod rebuild anyway. A
kernel-option change is a cache miss by design; a `ref` move is too.

`ref` accepts a tag, branch or commit. The role fetches it shallow first and falls
back to a full ref fetch, then to a whole-history fetch, for servers or commit
data that shallow fetch cannot serve; the resolved commit is checked out detached
and recorded in `artifacts/<artifact_id>/build-info/source.json`.

### Kernel options

`source.kernel` entries are merged into `.config` before `make defconfig`, then
asserted twice:

1. right after `defconfig` — a value the dependency graph dropped fails the build
   instead of silently shipping a kernel without it;
2. after the build — the generated kernel `.config` must carry
   `CONFIG_<name without CONFIG_KERNEL_>`; when BTF was requested, `vmlinux` must
   actually have a `.BTF` section.

`=n` is a real value: kconfig writes it back as `# CONFIG_<name> is not set`, and
that is the line both assertions look for. It matters for BTF, whose
`KERNEL_DEBUG_INFO_BTF` depends on `!KERNEL_DEBUG_INFO_REDUCED`, while
`KERNEL_DEBUG_INFO_REDUCED` defaults to `y` — a BTF variant therefore has to ask
for `CONFIG_KERNEL_DEBUG_INFO_REDUCED=n` as well.

`make defconfig` runs serially: the prerequisite checks it triggers share one
`$(TMP_DIR)`, so parallel jobs report the host compilers and tools as broken. The
first run on a fresh tree can still fail its aggregated prerequisite report
spuriously, so the build retries `defconfig` once and fails only if both attempts
fail.

This is the reason to build from source: the official ImageBuilder cannot change
the kernel configuration at all. A custom kernel changes the module ABI, so
`opkg install kmod-*` from the official release will not match the image; package
the modules you need instead.

## Modules

A module is enabled by listing any of its fields. Omitting the block, or leaving
an empty mapping such as `system: {}`, keeps the module disabled. Unknown keys
inside a module mapping are rejected by `make lint` / `make build`, so a typo
cannot silently disable a module. `disable-ipv6` is a plain boolean flag. A module
contributes first-boot overlay files; packages are explicit in `packages` unless
the module manages them (`statistics` does).

| Module | Description | Variant keys |
|--------|-------------|--------------|
| `system` | Hostname, timezone, and zonename | `hostname`, `timezone`, `zonename` |
| `disable-ipv6` | Disable IPv6, RA, and DHCPv6 on LAN/WAN | `true` / `false` |
| `statistics` | collectd + LuCI statistics configuration | `interfaces`, `sensors`, `irq`, `rrd_timespans` |
| `administration` | Initial root password, SSH listen interface/port | `root_password`, `ssh.interface`, `ssh.port` |

`network` is **not** a module: when a variant's `network` block is non-empty,
the role renders it with the same presence-based rule.

## Packages

`packages` is the operator's list written into the build configuration. Prefix a
name with `-` to remove it from the image (for example `-dnsmasq` +
`dnsmasq-full`); the role writes it as `# CONFIG_PACKAGE_<name> is not set`. A
source tree has no ImageBuilder `PACKAGES="-pkg"` syntax, so the explicit negative
is the mechanism, and after the build the role asserts the package is absent from
the target's `.manifest`. Duplicate entries are deduplicated; adding and removing
the same package is a hard error.

The `statistics` module owns its packages: it removes any `luci-app-statistics` /
`collectd-mod-*` entries from this list and adds the set matching the enabled
features. Listing them here is harmless.

The base set is in `inventory/host_vars/example/main.yml` and can be copied per
variant. The LuCI package-manager translation is explicit too:
`luci-i18n-package-manager-zh-cn` on 24.10+ (APK), `luci-i18n-opkg-zh-cn` on older
releases.

To resize a rootfs on the device, include `e2fsprogs` (`resize2fs`), `parted`, and
`fdisk` in `packages`.

### Ansible management

To let Ansible manage the built router with Python-based modules, include these
packages:

| Package | Purpose |
|---------|---------|
| `python3` | Full Python 3 + standard library (pulls `python3-light`, `python3-openssl`, `python3-urllib`, `python3-email`, `python3-logging`, `python3-ctypes`, ...) — the target interpreter for Ansible modules |
| `ca-bundle` | CA certificates for HTTPS/TLS verification (`get_url`, `uri`, feed downloads) |
| `openssh-sftp-server` | SFTP subsystem for Ansible's file transfer |
| `rsync` | `synchronize` module |

Optional, depending on the modules you use: `python3-cryptography` (crypto
modules), `git` (git module), `unzip`/`tar` (unarchive), `python3-pip` (pip
module). If flash is tight, `python3-light` plus the specific `python3-*`
subpackages (`python3-openssl`, `python3-urllib`, `python3-email`,
`python3-logging`) is a smaller alternative to the full `python3`.

`curl`, `vim-full`, `diffutils`, `bind-dig`, `nping`, `conntrack`, and `tcpdump`
are diagnostics; they are useful but not required by Ansible.

## Network

The `network` block configures LAN only; WAN (DHCP/PPPoE) is configured after
first boot.

```yaml
network:
  lan_ip: 192.0.2.1/24
  gateway: ""
  dns: []
  dhcp: true
```

| Key | Meaning |
|-----|---------|
| `lan_ip` | LAN static address (CIDR); empty ⇒ unchanged |
| `gateway` | LAN gateway; empty ⇒ unchanged (main router) |
| `dns` | LAN DNS list; empty ⇒ unchanged |
| `dhcp` | `false` disables the LAN DHCP server (AP/bypass); `true` leaves it on |

- **Main router**: set `lan_ip`, keep `gateway`/`dns` empty and `dhcp: true`.
- **AP / bypass router**: set `lan_ip`, `gateway`, `dns`, and `dhcp: false`.

## System

The `system` module sets the hostname, timezone, and zonename. The hostname
default is empty (keep the base image hostname); the timezone/zonename defaults
are `CST-8` / `Asia/Shanghai`.

```yaml
system:
  hostname: "openwrt-main"
  timezone: "CST-8"
  zonename: "Asia/Shanghai"
```

When the block is present, the module renders `etc/uci-defaults/91-system`. The
log levels (`conloglevel` / `cronloglevel`) are intentionally not part of the
build: they can be adjusted after boot.

## Statistics

The `statistics` module configures collectd/LuCI statistics; the collected
interfaces come from `interfaces`.

```yaml
statistics:
  interfaces: [br-lan, eth1]
  sensors: true
  irq: true
  rrd_timespans: [2hour, 1day, 1week, 1month, 1year]
```

| Key | Default | Meaning |
|-----|---------|---------|
| `interfaces` | `[]` | Interfaces collected by the interface plugin; empty disables it |
| `sensors` | `false` | Enable hardware sensors (`collectd-mod-sensors`) |
| `irq` | `false` | Enable IRQ statistics (`collectd-mod-irq`) |
| `rrd_timespans` | `[2hour, 1day, 1week, 1month, 1year]` | RRD history spans |

The `statistics` module installs its packages (and removes any duplicate
entries from `packages`): `luci-app-statistics` is always added, plus
`collectd-mod-sensors` when `sensors` is true and `collectd-mod-irq` when `irq`
is true.

RRD stands for **Round Robin Database**: a fixed-size circular time-series store
that RRDtool writes and LuCI graphs. RRD data writes to flash; enable it only if
you need the graphs.

## Custom overlay

For anything not covered by a module, set `custom_overlay` to a directory on the
control node. The role copies it over the image rootfs **after** the module
overlays, so it can add files or override module files:

```yaml
custom_overlay: "{{ inventory_dir }}/custom/{{ inventory_hostname }}/{{ build_variant.name }}"
```

Layout (per machine and variant; `inventory/custom/` is gitignored):

```text
inventory/custom/<machine>/<variant>/
├── etc/uci-defaults/90-custom     # chmod +x; runs on first boot
└── etc/config/...                 # plain files copied verbatim
```

Example `etc/uci-defaults/90-custom`:

```sh
#!/bin/sh
uci set system.@system[0].hostname='custom-router'
uci set network.lan.domain='example.lan'
uci commit system
uci commit network
```

The role preserves file modes, so `chmod +x` the uci-defaults script. Use a
separate directory per variant when the values differ. Custom files may contain
secrets; keep `inventory/custom/` out of version control.

For a reusable feature, add a module to the role instead (`module_names` +
`files/overlay/<name>` or `templates/overlay/<name>`); `custom_overlay` is for
operator-specific one-offs.

## Administration

The `administration` module renders the LuCI Administration settings:

```yaml
administration:
  root_password: ""     # empty = keep the system default; setting it bakes a secret
  ssh:
    interface: ""       # empty = listen on all; e.g. "lan"
    port: ""            # empty = default 22; e.g. "22"
```

- `root_password`: optional **initial** root password; the first-boot script
  sets it once, and you change it after login. Setting it bakes that initial
  password into the image; leave it empty to keep the system default.
- `ssh.interface`: binds dropbear to that interface; empty keeps the system
  default.
- `ssh.port`: sets `dropbear.Port`; empty keeps the default (22).

## Build resources

| Knob | Default | Effect |
|------|---------|--------|
| `source_min_free_gb` | `25` | Preflight refuses to start below this much free space on the volume holding `~/.cache/openwrt-builder` |
| `source.jobs` | `""` | Parallel make jobs; empty derives `min(vCPU, RAM-GB / 2)` from the reported RAM (pin it when the build host balloons) |
| `source.ccache` | `true` | Shared compiler cache under `~/.cache/openwrt-builder/ccache` (`CONFIG_DEVEL=y` + `CONFIG_CCACHE=y` + `CONFIG_CCACHE_DIR`) |
| `source.clean` | `false` | `make clean` before the build (keeps `dl/` and `ccache/`) |
| `docker_registry_mirrors` | `[]` | Docker Hub pull-through mirrors written into the build host's `daemon.json` |

A first full source build takes tens of minutes and roughly 30 GB
(`build_dir` + `dl/` + `ccache/`); an incremental `make world` after that takes
minutes. `make cache-clean PURGE=1` throws the source trees and `dl/` away, so the
next build is full again.

There is no package-mirror knob: a source build downloads from the sources the
tree itself configures. To mirror the git side, point `source.repo_url` at a
mirror of `openwrt/openwrt`; that is the only fetch the project performs beyond
`dl/`.

## Host security (SSH hardening)

`roles/sshd` optionally closes password login on the build machine and disables
root SSH, applied as a drop-in under `/etc/ssh/sshd_config.d/` — the running
`sshd_config` is never edited. It is OFF by default and runs at the end of
`make build` / `make deploy` (tag `sshd`).

| Knob | Default | Effect |
|------|---------|--------|
| `sshd_hardening_enabled` | `false` | Apply the hardening drop-in |
| `sshd_permit_root_login` | `"no"` | `no` disables root SSH; `prohibit-password` keeps root reachable by key |

It only proceeds when a key-only login survives it: the `iac` account's
public key (`host_iac_ssh_pubkey_file`) or, for a root-administered
host, root's key with `sshd_permit_root_login: prohibit-password`. The role
verifies the effective configuration (`sshd -T`), proves a key-only login from
the control node and rolls the drop-in back if either check fails.

## Firmware secrets

No firmware secrets are baked into the image. The WAN (PPPoE/DHCP) and root
passwords are set after first boot (or by a separate provisioning mechanism), so
the built artifact contains no credentials. There is no per-machine firmware
vault; connection credentials live in each machine's `host_vars/<machine>/vault.yml`. An
`administration.root_password` or a `custom_overlay` file is the exception the
operator chooses explicitly.

## Growth path

- **Another machine**: `make host-init HOST=<machine> HOST_IP=<addr>` and list its
  variants in `host_vars/<machine>/main.yml`.
- **Another variant on a machine**: append an entry to `build_variants` in that
  machine's `main.yml`; no role change.
- **Another target/profile**: change `source.target` and `source.profile`;
  a different target starts from a fresh source tree by design.
- **A custom module**: add `roles/buildroot/files/overlay/<name>/...` (static)
  or `templates/overlay/<name>/...` (templated), add `<name>` to `module_names`
  (presence-based mapping) or `module_flags` (a plain boolean) in
  `roles/buildroot/vars/main.yml`, and expose its variant keys. Its packages
  stay explicit in `packages`.
