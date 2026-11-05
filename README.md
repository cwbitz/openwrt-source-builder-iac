<div align="center">

# OpenWrt Source Builder IaC

Build customized OpenWrt firmware from source, declaratively.

One inventory host = one build machine; one variant = one pinned source build.

[![License](https://img.shields.io/badge/license-MIT-blue.svg?style=flat-square)](LICENSE)

</div>

Ansible IaC that compiles **OpenWrt from source** on a build host: pinned git ref,
feeds, kernel options (BTF/ftrace and friends), packages, first-boot overlay and
output artifacts — all declared in inventory, driven by `make build`.

The build host needs nothing but Docker: the entire buildroot toolchain lives in a
pinned image (`roles/buildroot/files/Dockerfile`), so the host stays clean and can
be shared with `openwrt-image-builder-iac` (the ImageBuilder project) without the
two contaminating each other.

## Why source instead of ImageBuilder

The official ImageBuilder repacks prebuilt binaries: it can add packages and
files, but it **cannot change the kernel configuration**. Building from source is
the only way to get things like `CONFIG_KERNEL_DEBUG_INFO_BTF` or
`CONFIG_KERNEL_DYNAMIC_FTRACE` into the image.

Two consequences to know before you switch:

- **Time and space**: a first full build takes tens of minutes and ~30 GB; the
  incremental `make image` step after that is minutes.
- **No official kernel modules after flashing**: a custom kernel changes the
  module ABI, so `opkg install kmod-*` from the official release fails. Package
  the modules you need into the image instead.

## Quick start

```bash
mise install                 # pinned toolchain (.tool-versions)
make install                 # collections + control-node dependencies
make host-init HOST=builder-1 HOST_IP=203.0.113.50
$EDITOR inventory/host_vars/builder-1/main.yml   # real source.ref/target/profile
make vault-init && make vault-edit HOST=builder-1
make host-bootstrap          # once per machine; creates the `iac` account
make build                   # every variant, machines in parallel
```

Artifacts land in `artifacts/<artifact_id>/targets/<target>/<subtarget>/` plus
`artifacts/<artifact_id>/build-info/` (resolved commit, feeds commits, the final
`.config`, and a hash of it), where `<artifact_id>` is
`<machine>-<variant>-sourcebuilder-<target>-<ref>`.

## Commands

| Target | What it does |
|--------|--------------|
| `make install` | Sole dependency entry point (toolchain, collections, sshpass, passlib, Keychain) |
| `make host-init HOST=… HOST_IP=…` | Scaffold `inventory/hosts.yml`, the group vars and `host_vars/<machine>/` |
| `make host-bootstrap` | Probe `iac → admin → root` and create/repair the `iac` automation account |
| `make build` | Build every variant (`EXTRA='--limit <machine>'` limits it) |
| `make info` | Query the pinned ref, target and profile list without building |
| `make deploy` | `host-bootstrap` then `build` |
| `make cache-clean [PURGE=1]` | Prune this project's builder caches (work + ccache; `PURGE=1` drops source trees and `dl/`) |
| `make lint` / `make test` | Local validation (same commands CI runs) |

## Layout

```
inventory/            hosts, group/shared vars, per-machine build_variants, knob schema
playbooks/            build / info / bootstrap / validate_knobs / host_keys / cleanup
roles/buildroot/      the source backend: preflight, tree+feeds, config, overlay,
                      container build, artifact fetch, metadata, info
roles/helpers/        vault read/write/generate helpers
roles/sshd/           opt-in key-only SSH hardening (off by default here)
scripts/              control-node helpers (agent runner, audit, privacy, vault sort)
```

## Sharing a machine with openwrt-image-builder-iac

Both projects may manage the same build host. The system layer (the `iac`
account and its sudoers file, the human `admin` account, SSH keys, sshd hardening
and Docker `daemon.json`) converges on identical desired state, so the same knobs
may be enabled in both: the second run re-derives the same values and reports no
change, and a diverging value (a different admin password or sshd policy) fails
loudly instead of silently overwriting the other project. This project only ever
writes below `~/.cache/openwrt-builder` (its own `source/`, `dl/`, `ccache/`,
`work/`) plus the control node's `artifacts/`.

## Docs

- [Architecture](docs/architecture.md) — components, configuration scope, dependency and secrets model, variant isolation.
- [Configuration](docs/configuration.md) — inventory layout, variant schema, the `source` block, packages, modules, build resources.
- [Build pipeline](docs/build-pipeline.md) — what a source build does, step by step, and what the artifacts contain.
- [Runbooks](docs/runbooks.md) — first-time setup, builds, caches, troubleshooting, manual verification.
- [AGENTS.md](AGENTS.md) — conventions, gotchas, testing.
