# Runbooks

Operational procedures for building OpenWrt firmware from source with
`openwrt-source-builder-iac`. For the design see
[architecture](architecture.md), for configuration see
[configuration](configuration.md), and for what a build does see
[build-pipeline](build-pipeline.md).

## 1. Prerequisites

| Requirement | Notes |
|-------------|-------|
| Control node | Where you run `make`; hosts the repo and the Ansible toolchain |
| Build host | A separate LAN machine (by default) running Docker Engine; SSH-reachable as the connection user, who must access the Docker daemon |
| sudo (or root) on the build host | Required on the first `make host-bootstrap` run to create the `iac` automation account |
| Docker Engine / Desktop | Must be running on the `builders` host(s); the build toolchain itself lives in the pinned image |
| GNU `make`, `perl`, `git` on the build host | `feeds update/install` runs outside the container, so the host needs these even though the compile does not |
| Disk | ~30 GB free on the volume holding `~/.cache/openwrt-builder` (the preflight floor is `source_min_free_gb`, default 25) |
| RAM | The effective job count is `min(vCPU, RAM-GB / 2)` from the RAM the guest reports, so a ballooned VM can silently drop to one job; 8 GB is a practical minimum, and `source.jobs` pins the count |
| python >= 3.8 | Control node; runs `ansible-core` and the `community.docker` modules |
| `ansible-core` + `python` | Pinned in `.tool-versions` (`mise install` / `asdf install`) |
| `community.docker` | Installed into `collections/` by `make install` |
| `openssl` | Control node; generates secret values (`make install`) |
| `sshpass` | Control node; password SSH during `make host-bootstrap` (`make install`) |
| `passlib` | Control node; `password_hash` for the automation user (`make install`) |
| Keychain 3 | Control node; **fallback** ssh-agent, ensured by `make install` |
| vault password | Created once by `make vault-init`; required to store/read generated secrets |
| docker Python SDK | On the **build host** (not the control node); checked by the role preflight |

`make install` is the sole dependency entry point and is idempotent; see
[architecture](architecture.md) for what it ensures and the docker-SDK boundary.

## 2. First-time setup

```bash
mise install
# One inventory host = one build machine; list its variants in the machine's main.yml.
# Extra tools in the build image: see docs/configuration.md ("The build environment
# and extra tooling"): per variant `source.build_image` plus an optional
# inventory/custom/build-image/Dockerfile.
make host-init HOST=builder-1 HOST_IP=203.0.113.50   # scaffolds group_vars + host_vars/<machine>/
# edit inventory/host_vars/builder-1/main.yml: source.ref/target/profile,
# kernel options, packages and settings
make host-init HOST=openwrt-src-prod-2 HOST_IP=203.0.113.51   # repeat for each machine
make install
make vault-init
make vault-edit HOST=builder-1      # per-machine SSH identity vault
make host-bootstrap                       # once; probe iac -> admin -> root
# or, to bootstrap and then build everything in one shot:
make deploy
```

Configuration is split by scope (machine-wide vs per-machine); see
[configuration](configuration.md). `hosts.yml`, the real
`group_vars/builders/main.yml`, and every `host_vars/<machine>/` directory are
gitignored; the committed `group_vars/all/main.yml` baseline and `*.example`
templates are tracked.

`host-init` copies the example verbatim, including the `<release-tag-or-commit>`,
`<target>/<subtarget>` and `<profile>` placeholders; validation refuses to build
until they are replaced.

### `host-bootstrap`

`make host-bootstrap` probes `iac → admin → root` on **every**
machine (or only `EXTRA='--limit <machine>'`) and uses the first identity that
works on each; when that is not `iac`, it
(re)creates the `iac` automation account (key-first, NOPASSWD sudo, the `docker`
group when present) and the optional human `admin` account. It self-heals a
rotated/deleted credential. With no `host_iac_ssh_pubkey_file`, an empty
`vault_host_iac_password` is generated into the machine's vault under the same
name. Passwords use a per-account random salt with `update_password: on_create`.
See [architecture](architecture.md) for the full model.

## 3. Build

```bash
make build                            # build every variant (machines in parallel)
make build EXTRA='--limit builder-1'  # single machine
make deploy                           # host-bootstrap, then build (fresh host to image)
```

`make build` targets `hosts: builders`; the role loops each machine's
`build_variants`, so machines build concurrently and a machine's variants
sequentially. The first full build compiles the toolchain, the kernel and every
package (tens of minutes); later runs reuse the cached source tree, `dl/` and
`ccache/`, so an incremental `make world` is minutes. See
[build-pipeline](build-pipeline.md) for the per-variant steps, container
orchestration and overlay rendering.

### Artifact retrieval

| Knob | Default | Effect |
|------|---------|--------|
| `artifacts_dir` | `<repo>/artifacts/<artifact_id>` | Control-node destination (always fetched); `<artifact_id>` = `<machine>-<variant>-sourcebuilder-<target>-<ref>-<digest>`, the digest covering the variant's kernel options, packages, profile, rootfs size and image name |
| `source_bin_dir` | `~/.cache/openwrt-builder/source/<key>/bin` | Builder-side `bin/` staging, removed before the next build |

Only `targets/<target>/<subtarget>/` is fetched (`bin/packages` is not); the
`artifacts/` directory is local-only and gitignored, and is created on demand.
Each build also writes `artifacts/<artifact_id>/build-info/source.json` (build image,
repo, ref, resolved commit, target, profile, jobs, kernel options, packages, feed
commits, config hash) and `config` (the final `.config`).

### Verify

```bash
ls -R artifacts
sha256sum -c artifacts/*/targets/*/*/sha256sums 2>/dev/null || true
cat artifacts/<artifact_id>/build-info/source.json
```

On a booted test device, confirm the kernel actually has what was requested:

```sh
test -e /sys/kernel/btf/vmlinux && echo BTF ok
test -d /sys/kernel/tracing && echo ftrace ok
```

The final `.config` that produced the image is in
`artifacts/<artifact_id>/build-info/config`; it carries the
`CONFIG_KERNEL_*` lines the build asserted.

Because the kernel is custom, `opkg install kmod-*` from the official release does
not match the image. Package the modules you need at build time.

### Incremental builds, caches and cleanup

| Target | Effect |
|--------|--------|
| `make cache-clean` | Removes `work/` and `ccache/` on the builder |
| `make cache-clean PURGE=1` | Also removes `source/`, `dl/`, `image/` — the next build is full again |

The source tree is keyed by `(target, ref, kernel options)`; changing a kernel
option is a deliberate cache miss. `dl/` and `ccache/` are shared across variants
and refs. These paths belong to this project alone: on a machine shared with
`openwrt-image-builder-iac`, its `work/` and `artifacts/` live under a different
project root and are never touched.

### Rollback / cleanup

The build writes only to the builder's `~/.cache/openwrt-builder` (work and
`bin/` scratch removed around each build) and the control node's `artifacts/`.
`artifacts/` contents are gitignored and can be removed freely; remove the
control-node caches (`.cache`, `collections`, `.ansible`) yourself. Nothing else
on either host is modified.

An interrupted (`Ctrl-C`) build is cleaned up automatically: the run's container
is force-removed with its anonymous volume and the builder-side scratch is
deleted. If the wrapper itself is killed with `SIGKILL`, run `make build` once
more (its cleanup removes the stale scratch) or remove the container by label:

```bash
ssh <builder> 'docker ps -aq --filter label=openwrt-builder=true | xargs -r docker rm -f -v'
```

## 4. Query source/target/profile info

```bash
make info
make info EXTRA='--limit builder-1'   # single machine
```

`make info` reports the pinned repo/ref, target, profile, build image and kernel
options without building. It lists the target's profiles from the **cached**
source tree, so on a fresh machine run `make build` once (or inspect
`target/linux/<target>/<subtarget>/profiles` in a checkout) before choosing
`source.profile`; when no tree is cached it says so explicitly. The dependency
helper chatter is silenced (`QUIET=1`) and the result is printed per variant.

## 5. Reproducibility

- Pin `source.ref` to a tag or commit; branches move.
- The resolved commit and every feed's `HEAD` are recorded in
  `build-info/source.json`, together with the final `.config` and its SHA256.
- The build image has one stable name (`cwbitz/openwrt-buildroot`): the role
  re-tags it whenever `roles/buildroot/files/Dockerfile` changes, and
  `build-info/source.json` records the Dockerfile's digest next to it, so an
  artifact still identifies the exact environment. A variant that needs extra
  tools sets `source.build_image` and is built from
  `inventory/custom/build-image/Dockerfile`.
- The cache key includes the kernel options, so two variants cannot silently share
  a tree with a different kernel configuration.

## 6. Secrets

- **No firmware secrets are baked** into the image: set the WAN (PPPoE/DHCP) and
  root passwords after first boot. An explicit
  `administration.root_password` or a `custom_overlay` file is operator-chosen and
  does end up in the artifact.
- **Connection credentials** are per machine (`inventory/host_vars/<machine>/vault.yml`).
- The vault password file lives outside the repo and is never committed;
  `make vault-edit` encrypts a plaintext vault before editing.
- Vault keys are kept in alphabetical order: `make vault-edit` re-sorts
  the top-level keys after the editor closes (comment-preserving) and re-encrypts
  only when the order changed. Hand edits should stay alphabetical too.
- Values are Jinja-templated and shell-quoted automatically, so special
  characters are safe.
- **The built image carries the secrets it was built with** — treat artifacts as
  sensitive.

## 7. Adding a module

1. Add overlay files under `roles/buildroot/files/overlay/<name>/...` (static)
   or `templates/overlay/<name>/...` (Jinja).
2. Add `<name>` to `module_names` (a presence-based mapping) or `module_flags`
   (a plain boolean) in `roles/buildroot/vars/main.yml`.
3. Reference its operator variables in the role defaults and
   `host_vars/example/main.yml`; add an `<name>:` mapping with its fields (or a
   boolean key for a flag module).
4. Add its structure check to `playbooks/validate_knobs.yml` and any new
   top-level knob to `inventory/knob_schema.yml`.
5. Add any packages the module needs. A module with package logic (like
   `statistics`) owns its set in `packages.yml`; otherwise list them in the
   variant's `packages`.
6. Run the internal `make lint`, then build with the module active.

For one-off operator settings (extra UCI, extra files), do **not** edit the
role: set the variant's `custom_overlay` to a directory under
`inventory/custom/<machine>/<variant>/`; the role copies it over the image
rootfs after the module overlays. See [configuration](configuration.md#custom-overlay).

## 8. SSH hardening

`make build` (and therefore `make deploy`) also applies `roles/sshd` at the end
when `sshd_hardening_enabled: true` is set in `group_vars/builders/main.yml`. It
closes password login, so it needs a key-only login that survives it — the
`iac` account's key (`host_iac_ssh_pubkey_file`) or root's key
with `sshd_permit_root_login: prohibit-password`. Run it alone with:

```bash
make build EXTRA='--tags sshd'
```

The role validates the drop-in (`sshd -t`), asserts the effective configuration
(`sshd -T`) and proves a fresh key-only login from the control node; on any
failure it restores the previous drop-in, reloads sshd and fails the run, so the
host keeps its old login policy. The provider console remains the recovery path.

## 9. Troubleshooting

| Symptom | Cause / fix |
|---------|-------------|
| `host-bootstrap`: "no working identity" | Configure the connection passwords in the machine's vault and/or the key paths in `group_vars/builders/main.yml`, or load a key into the SSH agent. If only `root` works, set the root password (or a root private key) |
| `build`/`info` cannot connect as `iac` | Run `make host-bootstrap` once; the effective password is `vault_host_iac_password` (generated when empty), else set `host_iac_ssh_privkey_file`/pubkey |
| Need the auto-generated password | `make vault-view` |
| `build` fails: vault password file missing | Run `make vault-init` |
| `host-bootstrap` user tasks fail (sudo/become) | The probed identity needs sudo or root; the first run must reach the host as `root` or a sudo-capable `admin` |
| "needs source.ref, source.target … and source.profile" | Replace the example placeholders; `make host-init` copies them verbatim |
| "cannot fetch '<ref>'" | Check the tag/branch/commit and network access; for a slow `git.openwrt.org`, set `source.repo_url` to a mirror |
| preflight: "only N GB free" | Free space on the volume holding `~/.cache/openwrt-builder`, or lower `source_min_free_gb` if you know the host has room elsewhere |
| preflight: "Docker CLI not found" | Install Docker and ensure it is on PATH on the builder |
| preflight: "Cannot reach the Docker daemon" | Start Docker; ensure the connection user can access the socket (the `iac` user joins the `docker` group when present) |
| preflight: "Docker Python SDK" | Install `docker` for the build host's Python: `<python> -m pip install docker` |
| "dropped by defconfig: CONFIG_…" | A requested kernel option cannot survive `defconfig` on this target (missing dependency/platform support). Remove it or fix the dependency; shipping without it is refused on purpose |
| "CONFIG_… missing from …/linux-…/.config" | Same contract, checked again after the build; the kernel config did not carry the option |
| "vmlinux carries no .BTF section" | BTF was requested but the toolchain/kernel did not produce it; check the build environment and the toolchain (dwarves/pahole) |
| "package removed by the variant is still in the image" | The `-pkg` write did not take effect (a profile dependency pulls it back). Inspect `.config` and the generated `.manifest` |
| "Packages are both added and removed" | Remove the `-pkg`/`pkg` conflict in a variant's `packages` |
| "Duplicate overlay target files across modules" | Two active modules ship the same file path; rename or disable one |
| A module does not run | The block is missing or empty; list at least one of its fields (a flag module needs `true`) |
| `statistics` packages | The module adds `luci-app-statistics` + `collectd-mod-sensors`/`collectd-mod-irq` itself; entries in `packages` are removed/deduped |
| WAN is not configured | The image intentionally leaves WAN (DHCP/PPPoE) unset; configure it after first boot |
| Build hangs at "Building package index..." / fakeroot | The container `nofile` ulimit is unbounded and fakeroot's `faked` spins; the role pins `nofile:65536:65536`, keep it bounded |
| Build is far slower than before | The cache key changed (ref, target or kernel options); a full rebuild is expected. Check `~/.cache/openwrt-builder/source` |
| `Jobs 1` in the build plan / a build far slower than the vCPU count suggests | The guest reports ballooned-down RAM, so `min(vCPU, RAM-GB/2)` collapses to 1. Pin the variant's `source.jobs`, or stop the build host from reclaiming memory (`memory_floating`) |
| Out of disk during a build | `dl/`, `ccache/` and the tree grow together; `make cache-clean PURGE=1` resets, then the next build is full |
| `apk add kmod-*` fails: `unable to select packages: kernel-…: breaks: kmod-…[kernel=…]` | The image's kernel ABI hash is not the release build's, so the official kmods do not belong to this kernel. Add the module to the variant's `packages` and rebuild, or point apk at this build's own `bin/targets/.../packages`. See `docs/build-pipeline.md` ("Kernel and module ABI") |
| Artifacts not in the control node's `artifacts/` | The fetch failed; inspect the fetch task. The builder keeps its `bin/` until the next build, so a re-run or a manual `scp` can recover it |
| `--tags sshd` fails at "Prove a key-only login" | The chosen identity has no authorized key yet: set `host_iac_ssh_pubkey_file` (or root's) and run `make host-bootstrap`, or unset `sshd_hardening_enabled` |
| Cannot reach a builder over SSH | Fix its `ansible_host` in `hosts.yml`; the connection identity comes from the committed `group_vars/all/main.yml` |

## Manual verification on a disposable host

The cases `tests/` cannot cover: they need a real build machine, the build
container and destructive steps. Run them on a disposable machine with its own
inventory and vault password file, read-only checks first, destructive ones last,
and evidence under `/tmp/opencode/test-run-<stamp>/artifacts/<case>/`, redacted.
The expected result is the contract — a deviation is a defect to record, not a
reason to skip.

### Bootstrap and first build

- [ ] `make host-init` scaffolds `inventory/hosts.yml` and `host_vars/<machine>/`;
      the placeholders are refused until replaced.
- [ ] `make host-bootstrap`: `iac` created with passwordless sudo and the configured
      keys, Docker Engine present; a second run changes nothing.
- [ ] `make build` builds every declared variant and fetches the images into the
      control node's `artifacts/<artifact_id>/targets/...`; `build-info/source.json`
      names the resolved commit, the feeds and the final config hash.
- [ ] The build container runs as the invoking uid/gid: the source tree and
      artifacts are not root-owned.
- [ ] A second `make build` reuses the cached tree and `dl/`, runs incrementally
      and converges on the same output.
- [ ] Two variants on one machine build in sequence without sharing or clobbering
      their `work/<build-id>` directories.

### Kernel, packages and overlay

- [ ] Requested kernel options (`BTF`, `ftrace`, ...) are present in the built
      kernel; `/sys/kernel/btf/vmlinux` and `/sys/kernel/tracing` exist on the
      device.
- [ ] A deliberately unsupported kernel option fails the build right after
      `defconfig` (the "dropped by defconfig" message) instead of shipping.
- [ ] Each variant's packages are reflected in the image; a `-pkg` removal is
      absent from `.manifest`.
- [ ] `disabled_services` are disabled after first boot; the generated
      `99-disabled-services` script runs once.
- [ ] `make info` reports the target/profile without starting a build.
- [ ] The artifact's package set matches the equivalent ImageBuilder variant
      (`.manifest` compare), or the difference is explained.

### Caches, secrets and security

- [ ] Changing only a kernel option produces a new cache key and a full rebuild;
      changing nothing reuses the tree.
- [ ] `make cache-clean` removes `work/` + `ccache/`; `PURGE=1` also drops
      `source/`, `dl/`, `image/`, and the next build is full.
- [ ] No firmware secrets are in the artifacts or the tree: a built image carries
      no WAN/root password.
- [ ] Each machine vault is `0600` and encrypted; `make vault-view` reads it
      and `-vvv` shows no value.
- [ ] With `sshd_hardening_enabled`: password login refused, keyed login works, and
      a missing authorized key blocks the role before it writes the config.
- [ ] `make lint` and `make test` are green.
- [ ] `AGENTS.md`, `docs/` and this runbook match observed behaviour; a
      contradiction is recorded as a defect.
