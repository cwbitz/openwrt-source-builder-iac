# AGENTS.md

Ansible IaC that builds customized **OpenWrt firmware from source** on a build
host. One inventory host = one build machine; each machine builds one or more
**variants** (one `(ref, target, profile, kernel options, settings)` each),
listed in its `host_vars/<machine>/main.yml` as `build_variants`.

Entry playbooks: `playbooks/bootstrap.yml` (once, provisions the `iac` automation
account), `playbooks/build.yml` (builds every variant, plus opt-in SSH hardening)
and `playbooks/info.yml` (read-only ref/target/profile query). The build logic
lives in `roles/buildroot/`; the opt-in key-only SSH hardening lives in
`roles/sshd/`.

## Commands

- `make help` — aligned target list; overridable variables (`HOST`, `EXTRA`,
  `PURGE`) are shown inline. `help` is listed last.
- `make install` — the sole dependency entry point (toolchain, collections,
  `sshpass`, `passlib`, Keychain). Every other target depends on it.
- `make host-init HOST=<name> HOST_IP=<addr>` — scaffold `inventory/hosts.yml`,
  `group_vars/builders/main.yml` comment-free from the examples, plus
  `host_vars/<name>/` (with a `build_variants` list); add `<name>` under
  `builders.hosts` with `HOST_IP` as `ansible_host`; encrypt the vault (dir
  `0700`, file `0600`). Re-running does not overwrite existing files.
- `make host-bootstrap` — once: probe `iac → admin → root` on every machine and
  create/repair the `iac` automation account (key-first, NOPASSWD sudo, `docker`
  group) plus the human `admin` account when `create_admin_account` is set.
  `REFRESH=1` refreshes host keys first.
- `make deploy` — one-click: `host-bootstrap` then `build`.
- `make build` / `make info` — build every variant (machines in parallel) / query
  the pinned ref and profile list; limit to one machine with
  `EXTRA='--limit <machine>'`.
- `make cache-clean [PURGE=1]` — prune this project's caches on the builder
  (`work/` + `ccache/`; `PURGE=1` also drops `source/`, `dl/`, `image/`).
- `make hostkey-init` — refresh every machine's SSH host key in the control
  node's `known_hosts` (drop the stale entry, re-add).
- `make vault-init` / `vault-edit HOST=<machine>` / `vault-view [HOST=<machine>]`
  — the per-machine connection vault; plaintext is encrypted first and the keys
  are re-sorted alphabetically after editing. With a single machine `HOST` is
  implied; `VAULT_FILE=<path>` overrides the path.
- Internal (hidden from `make help`): `lint` (syntax-check + `ansible-lint` +
  `yamllint` + `shellcheck` + the operator-example audit + the knob validation +
  `scripts/check_repo_privacy.py` + `gitleaks`; run by CI), `make test` (helper
  unit suite) and the `ensure-*` / `agent-*` / `keychain-*` / `run-playbook`
  helpers. `make build` and `make lint` assert every operator knob against
  `inventory/knob_schema.yml` through `playbooks/validate_knobs.yml`.
- `EXTRA` is appended to every `ansible-playbook` invocation, e.g.
  `make build EXTRA='--limit builder-1'`.

## Variant schema

Shared with the ImageBuilder project (`packages`, `system`, `network`,
`administration`, `statistics`, `custom_overlay`, `extra_firmware_name`,
`disabled_services`, `rootfs_partsize`), plus one source-only block that also
carries the device profile:

```yaml
build_variants:
  - name: main
    source:
      ref: v25.12.6                 # [REQUIRED] tag/branch/commit-ish (pinned)
      target: mediatek/filogic      # [REQUIRED] <target>/<subtarget>
      profile: glinet_gl-mt6000     # [REQUIRED] profile of that target
      repo_url: https://git.openwrt.org/openwrt/openwrt.git
      update: false                 # git fetch before building
      jobs: ""                      # empty => min(nproc, RAM-GB / 2)
      clean: false                  # make clean (keeps dl/ and ccache/)
      ccache: true                  # shared ccache directory
      kernel:                       # asserted again after `make defconfig`
        - CONFIG_KERNEL_DEBUG_INFO_BTF=y
        - CONFIG_KERNEL_DEBUG_INFO_REDUCED=n
```

The build environment is one image (`cwbitz/openwrt-buildroot`, built from
`roles/buildroot/files/Dockerfile`): the role re-tags it when that file changes
and records the Dockerfile digest in `build-info/source.json`. A variant that
needs different tools sets `source.build_image`; the role then builds that image
from `inventory/custom/build-image/Dockerfile` when that gitignored file exists,
otherwise it uses or pulls the reference as-is. Nothing is re-tagged under a
name it did not build.

- **`source.kernel`** entries must be `CONFIG_<NAME>=y|m|n`. A value dropped by
  `make defconfig` fails the build instead of silently shipping a kernel without
  it (that is how BTF/ftrace would otherwise vanish). `=n` is matched against the
  `# CONFIG_<NAME> is not set` line kconfig writes back; BTF needs it for
  `KERNEL_DEBUG_INFO_REDUCED`, which defaults to `y` and would otherwise drop
  `KERNEL_DEBUG_INFO_BTF`.
- **`packages`** with a `-` prefix is written as `# CONFIG_PACKAGE_x is not set`
  (the source tree has no ImageBuilder `PACKAGES="-pkg"` syntax), and the build
  asserts the package is absent from the generated `.manifest`.
- **`disabled_services`** has no make variable in a source build: the generated
  overlay disables the services at first boot.
- **Cache key** = `<target>-<ref>-<sha1(kernel options)[0:8]>`, so two variants
  that differ only in kernel options do not share a tree (a kernel change forces
  a full kernel + kmod rebuild anyway).

## Build pipeline (`roles/buildroot`)

Task slices, individually selectable with `--tags`
(`preflight|modules|packages|setup|config|overlay|build|fetch|record`):

1. **preflight** — asserts `source.ref`/`source.target`/`source.profile` (and no
   placeholders), Docker CLI/daemon/SDK on the host, the free-space floor
   (`source_min_free_gb`) and derives the effective job count.
2. **modules / packages** — same semantics as the ImageBuilder project.
3. **setup** — ensures the pinned build image (rebuilt when the Dockerfile
   changes), clones/fetches the tree into `~/.cache/openwrt-builder/source/<key>`,
   `feeds update -a` (new tree or `source.update`) and `feeds install -a`.
4. **config** — renders `.config` (target/subtarget/device, kernel options,
   packages and removals, `CONFIG_DOWNLOAD_FOLDER`, `CONFIG_CCACHE`).
5. **overlay** — renders the module uci-defaults + `custom_overlay` and installs
   them as `<src>/files/`.
6. **build** — runs the pinned container (`download` → `world`, which also
   builds the images), then
   asserts the requested kernel options survived, the kernel config carries them,
   `.BTF` exists when requested, and removed packages are absent from `.manifest`.
7. **fetch** — copies only `bin/targets/...` to
   `artifacts/<artifact_id>/targets/...` (never the `bin/packages` tree).
   `artifact_id` is `<machine>-<variant>-source-<target>-<ref>-<digest>`, the
   digest covering the variant's kernel options, packages, profile, rootfs size
   and image name, so a changed variant gets its own directory.
8. **record** — writes `artifacts/<artifact_id>/build-info/{source.json,config}`.

The build container runs as the invoking host uid/gid (no root-owned tree, no
`chown -R` pass) and must keep `--ulimit nofile:65536:65536`: fakeroot's `faked`
walks every fd up to `RLIMIT_NOFILE` on startup and hangs on a near-infinite
limit.

## Sharing the machine with `openwrt-image-builder-iac`

The system layer is owned by whichever project bootstraps first and must be
configured identically in both, so a later run is a no-op:

| Resource | Owner | This project |
|---|---|---|
| `iac` account (uid/gid 6000, NOPASSWD sudo, docker group) | first bootstrap | same desired state |
| `/etc/sudoers.d/90-openwrt-builder-iac` | first bootstrap | **keep this exact name** |
| `vault_host_iac_password` | — | leave empty in both; use a key |
| human `admin` account | either project | same `vault_host_admin_password`; a mismatch fails loudly |
| `/etc/docker/daemon.json` | either project | same `docker_registry_mirrors`; merged idempotently |
| sshd hardening drop-in | either project | same filename + `sshd_permit_root_login`; a mismatch fails loudly |
| `~/.cache/openwrt-builder/source|dl|ccache` | this project | never removed by others |
| `~/.cache/openwrt-builder/work|artifacts` | image-builder project | not touched |

### Both projects may set the same knobs to true

`create_admin_account` and `sshd_hardening_enabled` are safe to enable in **both**
projects, because each one converges on the same declared state and refuses to
guess when it does not:

- **`iac` account** — the second run probes as `iac` and skips account creation;
  key authorization and the sudoers file are written to the same paths with the
  same content (`changed=0`).
- **`admin` account** — the password hash is re-derived with the *existing* salt
  and rounds (the 5000-round default when the hash carries no `rounds=` field,
  which is what cloud-init writes), so the same vault value produces a
  byte-identical hash. Before
  writing, `host-bootstrap` asserts that either the account is new, the existing
  hash already matches this project's vault, or the hash uses an algorithm it
  cannot compare (warned). It also refuses to *generate* a password for an
  account that already exists. A divergent vault therefore fails with an
  actionable message instead of silently rotating the password away from the
  other project.
- **sshd hardening** — the role renders one shared drop-in
  (`00-sshd-hardening.conf`). If the existing managed file carries a different
  `PermitRootLogin` policy, the run fails instead of overwriting it.
- **`daemon.json`** — mirrors are merged into the existing file, so the same
  `docker_registry_mirrors` list on both sides is a no-op.

The checks are deliberately *compare-then-refuse*, not *exists-then-skip*: a pure
"already there, do nothing" probe would make the first writer permanent and stop
repairing drift (a half-provisioned account, a stale drop-in).

Idempotency is necessary but not sufficient: after a fresh VM the **first**
bootstrap must come from a project that holds `admin`/`root` credentials. Once
the `iac` account and its key exist, every other run (from either project) probes
as `iac` and converges.

## Secrets and local-only files

- `inventory/group_vars/all/main.yml` — committed baseline: the SSH connection
  identity wiring, the plugin interface derived from it, and the per-machine
  vault path.
- `inventory/group_vars/builders/main.yml` — machine-wide operator overrides,
  local only; template is `main.yml.example`.
- `inventory/host_vars/<machine>/vault.yml` — that machine's connection
  passwords, local only; `make host-init` scaffolds it from
  `inventory/host_vars/example/vault.yml.example` (mode 0600).
- Vault password file:
  `~/.config/projects/openwrt-source-builder-iac/ansible_vault_password`
  (pointed to by `ansible.cfg`, generated by `make vault-init`); never committed.
- No firmware secrets are baked into the image: WAN and root passwords are set
  after first boot. The built image carries whatever was built with it — treat
  fetched artifacts as sensitive.

## Gotchas

- The first build is expensive (tens of minutes, ~30 GB). `make cache-clean
  PURGE=1` throws away the source trees and `dl/`, so the next build is full again.
- `dl/` and `ccache/` are shared across variants and refs; `source/<key>` is not.
- A commit that changes a requested kernel option changes the cache key, so it
  starts from a fresh tree by design.
- The `.manifest` package-compare is the parity check against the ImageBuilder
  variant; keep it in the runbook when changing the package logic.
- `ansible.cfg` sets `vault_password_file`; ansible-core 2.19 hard-fails when it
  is missing, so CI generates a throwaway file before lint/syntax-check.

## Testing and verification

- **Automated** — `make lint` (syntax check, `ansible-lint`, `yamllint`,
  `shellcheck`, the knob-schema/example audit, the host-key and repository-privacy
  checks, `gitleaks`) and `make test` (helper suite under `tests/`). Both run in
  CI on every change. CI never runs a real build.
- **Manual** — a disposable build host checklist: first full build, a second run
  at the same desired state, an incremental `make world`, and the on-device BTF
  check (`/sys/kernel/btf/vmlinux`, `/sys/kernel/tracing`). Live hosts stay
  read-only.
