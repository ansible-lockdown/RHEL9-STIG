# Molecule Quick Start - RHEL9-STIG

> Audience: anyone new to molecule testing with this Ansible Lockdown role. Read top-to-bottom the first time; come back to the reference tables later.

## Two scenarios live here: `default/` vs `ubi/`

This `molecule/` directory ships two scenarios. Pick the right one before running:

| | `molecule/default/` (canonical) | `molecule/ubi/` (smoke test) |
|---|---|---|
| **When to use** | Every QA cycle, before merging/pushing. This is the **gating scenario**. | Occasional sanity check against Red Hat's upstream UBI image (e.g. confirm role still applies on the registry image, not just on the Rocky variant). |
| **Container image** | `rockylinux/rockylinux:9-ubi-init` | `redhat/ubi9-init:latest` |
| **Container name** | `rhel9-stig-qa` | `rhel9-stig-ubi` |
| **`audit_git_version`** | not pinned in the scenario - derived from `benchmark_version` via `vars/audit.yml`; override at run time with `--extra-vars` | same - derived from `benchmark_version`, so it cannot drift |
| **`rhel9stig_disruption_high`** | `true` (exercises full code path) | `false` (conservative) |
| **`fetch_audit_output`** | `true` (auto-copies audit JSONs out to `_temp_fetched_audits/`) | not set (audit results stay inside the container; `docker cp` them out manually) |
| **`prepare.yml` package set** | Full (audit, aide, chrony, rsyslog, logrotate, cronie, acl, kmod, dnf-plugins-core, python3-libselinux, python3-policycoreutils, python3-dnf-plugin-versionlock, tmux, git, procps-ng, openssl-pkcs11, opensc) | Subset; some packages aren't in UBI repos so installs run with `ignore_errors: true` |
| **Maintenance** | Kept in lock-step with the role and audit repos every cycle | Updated less often; intentionally drift-tolerant |

**Rule of thumb:** if you don't know which to use, use `default/`. `ubi/` is for cases where you specifically want to validate against Red Hat's `redhat/ubi9-init` registry image rather than the Rocky variant.

The rest of this document focuses on the `default/` scenario. To run the `ubi/` scenario instead, swap `molecule destroy` -> `molecule -s ubi destroy` (and the same `-s ubi` flag on `converge`, `verify`).

## What this scenario does

Molecule spins up a throwaway Rocky 9 Docker container, applies the **whole** Ansible Lockdown RHEL 9 STIG role against it, then runs the paired goss audit (from the `RHEL9-STIG-Audit` repo) to see how many controls passed and how many still fail. It's the gating test you run before merging or pushing benchmark changes.

End-to-end you get:
1. A clean container booted into systemd.
2. Pre-remediation audit (baseline): records which controls are already failing before the role runs.
3. Role converge #1: applies all the STIG remediations.
4. Role converge #2: re-runs to confirm idempotency (zero changed tasks the second time).
5. Post-remediation audit: records the after-picture.
6. Pre and post audit JSON summaries automatically copied out to a directory beside the role for review.

If steps 3 and 4 both report `failed=0` and step 6 shows the post-audit failure count substantially lower than the pre-audit count, the role is shippable.

## Prerequisites

| Need | Why | How to check |
|---|---|---|
| Docker (Desktop or Engine), running | Molecule uses Docker to create the test container | `docker info` returns without error |
| Python 3.10+ | Ansible / Molecule are Python tools | `python3 --version` |
| Ansible venv with `ansible-core >= 2.19`, `molecule`, `molecule-plugins[docker]`, `docker`, `passlib` | Runtime deps for the test | `pip list \| grep -E 'ansible|molecule'` |
| `git` on the controller | Audit content is cloned from `RHEL9-STIG-Audit` | `git --version` |
| Local clones of **both** `RHEL9-STIG` AND `RHEL9-STIG-Audit` (or network access so the audit repo can be cloned at runtime) | The role pulls audit goss content from the audit repo during converge | `git -C <path-to>/RHEL9-STIG-Audit status` |

One-time venv setup (skip if you already have one):

```bash
python3 -m venv <path-to-your-ansible-venv>
source <path-to-your-ansible-venv>/bin/activate
pip install 'ansible-core>=2.19' 'molecule>=24' 'molecule-plugins[docker]' docker passlib
```

## Quick start

```bash
# Every time
source <path-to-your-ansible-venv>/bin/activate
cd <path-to>/RHEL9-STIG

# Full gating pair
molecule destroy && molecule converge && molecule converge && molecule verify
```

The whole sequence takes 15-25 minutes on a modern laptop. Network is needed (clone of audit content + dnf installs) for the first run; subsequent runs reuse the cached Docker image.

## What each command does

| Command | What happens |
|---|---|
| `molecule destroy` | Removes any leftover container from a previous run. Safe to run on a clean system. |
| `molecule converge` (first) | (a) pulls / starts the `rockylinux/rockylinux:9-ubi-init` container, (b) runs `prepare.yml` to install supporting packages, (c) clones the audit content from `RHEL9-STIG-Audit`, (d) runs the pre-remediation audit, (e) applies the full role, (f) runs the post-remediation audit, (g) fetches both audit JSONs to your local `_temp_fetched_audits/` folder. |
| `molecule converge` (second) | Re-runs the role against the already-remediated container. The Ansible `PLAY RECAP` should show `changed=0` (or a small handful of acceptable re-renders) - that's the **idempotency check**. |
| `molecule verify` | Final assertions defined in the scenario (light by default for this role; the real validation is in the audit JSONs). |

## Where the audit results land

After `molecule converge` completes, look at:

```
<working-tree>/_temp_fetched_audits/
  rhel9-stig-qa-RHEL9-STIG-v<ver>_pre_scan_<timestamp>.json
  rhel9-stig-qa-RHEL9-STIG-v<ver>_post_scan_<timestamp>.json
```

The JSONs are goss output - each has a `results: []` array where every entry has a `successful: true|false` field. Compare pre vs. post:

```bash
# How many controls passed before remediation?
jq '[.results[] | select(.successful==true)] | length' <pre_scan>.json

# How many passed after?
jq '[.results[] | select(.successful==true)] | length' <post_scan>.json
```

If the second number is significantly higher than the first, the role is working as intended.

## How to read `PLAY RECAP`

After each converge, Ansible prints a one-line summary like:

```
PLAY RECAP ********************************************************
rhel9-stig-qa : ok=238  changed=73  unreachable=0  failed=0  skipped=547  rescued=0  ignored=0
```

| Counter | What it means |
|---|---|
| `ok` | Tasks that ran and finished successfully (no state change needed, or already in desired state) |
| `changed` | Tasks that modified the system (installed a package, edited a file, etc.) |
| `failed` | Tasks that errored out. **MUST BE 0** for the run to be considered passing. |
| `skipped` | Tasks gated off by `when:` conditions (often because `system_is_container: true` skips container-incompatible work) |
| `rescued` / `ignored` | Block-level error handling - typically 0 |

**Idempotency expectation:** the second `converge` should show `changed=0` (or near zero) - the system was already in the desired state. A high `changed` count on the second run means a task is not idempotent and needs fixing.

## Why some audit failures are expected (even after the role passes)

The post-scan will still show some controls failing. Most fall into three buckets that aren't role bugs - they're inherent to containerized testing:

1. **SSH-config controls** (banner, ciphers, KexAlgorithms, MACs, idle timeouts, login grace time): these check `/etc/ssh/sshd_config`, which doesn't exist because `openssh-server` is intentionally NOT installed (running sshd inside Docker is an anti-pattern; containers use `docker exec` for shell access). The role's `when: rhel9stig_ssh_required` gate handles this correctly on real hosts.

2. **User-state controls** (FIPS-hashed passwords, password lifetime, dotfile audit): need real interactive users (UID >= 1000) with shadow entries and home directories. Containers only have system accounts (`chrony`, `polkitd`, `tss`, etc.), so the controls can't satisfy their preconditions.

3. **Kernel / mount controls**: any control gated by `not system_is_container` is intentionally skipped during remediation (auditd, FIPS, kernel sysctls, mount options) because containers share the host kernel. Audit still runs the goss tests, so they report as failing.

None of these warrant fixing the role. They're the expected delta between a containerized test and a real RHEL 9 host.

## Common gotchas

| Symptom | Likely cause | Fix |
|---|---|---|
| `molecule converge` errors with "container not running" | Stale container from a previous interrupted run | Run `molecule destroy` first, then retry |
| Pre-task fails with "Failed to download metadata" | UBI image can't reach Red Hat repos, or no network | Check Docker network; some packages may be unavailable in UBI - that's expected (see `ignore_errors` in `prepare.yml`) |
| Converge #2 fails on a task that passed in #1 | Ansible 2.19+ struct-vs-string type-check tripping a `when:` clause that's "lucky" on the first run | Inspect the offending task's `when:` - look for quoted-string-as-boolean bugs |
| `Conditional result (True) was derived from value of type 'str'` | Ansible 2.19+ rejects when-clauses whose final value is a non-boolean string. Common bug: an `or "<some expression>"` where the RHS got wrapped in quotes by accident | Remove the quotes; the RHS should be a bare Jinja expression |
| `verify` logs `Executed: Missing playbook` and exits 0 | The scenario has no `verify.yml`, so the ansible verifier asserts nothing and still reports green - a silent no-op, not a pass | Add a `verify.yml` to that scenario. Both `default/` and `ubi/` have one; `ubi/verify.yml` imports the default assertions so there is a single source of truth |
| `verify` passes but asserts almost nothing | Expected in a container - nearly all of `verify.yml`'s assertions are gated to a real host so they skip rather than false-fail | See "What `verify.yml` actually covers in a container". Judge a run on converge `failed=0` and the goss deltas, not on a green `verify` |
| `create` fails with "image ... was found but its platform ... does not match" | You set `MOLECULE_DOCKER_PLATFORM` to a non-native arch while the native image is still cached, and `pre_build_image: true` reuses the cache instead of pulling | `docker rmi <image>` first - see "Choosing the container architecture" |

## Choosing the container architecture

Both scenarios set `platform: "${MOLECULE_DOCKER_PLATFORM:-linux/arm64}"`. Leave the variable unset and you get `linux/arm64`, which is what you want on Apple Silicon - that is the default the local gating runs have always used. Export it to pick a different arch:

```bash
MOLECULE_DOCKER_PLATFORM=linux/amd64 molecule converge
```

Both images (`rockylinux/rockylinux:9-ubi-init`, `redhat/ubi9-init`) are multi-arch, so either value pulls a native image.

Forcing the non-native arch locally runs under emulation and is slow - worth doing only to reproduce a CI failure. It also needs one extra step. The scenarios set `pre_build_image: true`, so molecule reuses whatever is already in your local image cache and only pulls when nothing is there. If you already have the arm64 image cached and ask for `linux/amd64`, container creation fails outright:

```
Error creating container: 404 ... image with reference rockylinux/rockylinux:9-ubi-init
was found but its platform (linux/arm64/v8) does not match the specified platform (linux/amd64)
```

Drop the cached image first, then create:

```bash
docker rmi rockylinux/rockylinux:9-ubi-init
MOLECULE_DOCKER_PLATFORM=linux/amd64 molecule converge
```

CI never hits this - a fresh runner has an empty image cache, so the pull resolves to the requested arch on the first try. Re-pull the native image (`docker pull --platform linux/arm64 rockylinux/rockylinux:9-ubi-init`) when you go back to local runs.

## CI (GitHub Actions)

`.github/workflows/molecule.yml` runs both scenarios on GitHub-hosted `ubuntu-latest` runners, in a `fail-fast: false` matrix, on pull requests to `latest` and `benchmark*`, on pushes to `latest`, weekly, and on demand. Because the runners are amd64, the workflow exports `MOLECULE_DOCKER_PLATFORM=linux/amd64`. The workflow file is intentionally byte-identical to the one on the V2R9 branch.

This is a fast, secret-free gate that runs in front of the tofu/EC2 pipelines - it does not replace them. Three things about it differ from the local ritual above and are deliberate:

- **It runs `create` -> `converge` -> `verify` -> `destroy`, not `molecule test`.** `molecule test` includes an `idempotence` step, and converge #2 for this role is a known non-zero-change run (the `rhel9stig_disruption_high`-gated 411090 pam_faillock/authselect settle, 252060 `/etc/aliases`, plus harness tasks that always report changed). Gating CI on it would be red for reasons that are not regressions. Keep running the double converge locally - that is still where idempotency drift gets caught.
- **Goss results do not fail the build.** The post-remediation audit JSONs are uploaded as a `molecule-audit-<scenario>` artifact for inspection. See "Why some audit failures are expected" above.
- **Where the CI signal actually comes from.** The gate is converge finishing `failed=0`, read alongside the goss deltas in the artifact. `verify.yml` is a narrow regression check layered on top, not the substance of the gate - see the next section.

Note that `ansible-core` ships no bundled collections, so the workflow installs `community.docker` and `ansible.posix` explicitly - the molecule docker driver needs both, and `collections/requirements.yml` does not carry `community.docker`.

## What `verify.yml` actually covers in a container

`verify.yml` is a targeted regression check, not a broad conformance test. Most of it is written to fire on a real host and skip cleanly in a container rather than false-fail, so under molecule the great majority of it skips: the audit-rule and 653110 assertions need auditd to be managed (it is not, per `vars/is_container.yml`), and 255120 needs SSH host keys, which a container does not have. Judge a run on converge `failed=0` and the goss deltas; a green `verify` step is close to no evidence on its own.

**This branch's `verify.yml` encodes V2R8 semantics and is deliberately the inverse of the V2R9 branch's on two controls.** Do not copy either file across branches wholesale:

| Control | V2R8 (this branch) asserts | V2R9 asserts |
|---|---|---|
| 654215-654255, 654097 | watch-style rules (`-w <path> -p wa -k <key>`) are **present** | syscall-form rules are present, watch-style **removed** |
| 211045 | drop-in at the **suffixless** path; the `.conf` path must not exist | drop-in at the **`.conf`** path; suffixless must not exist |
| 653110 | audit config modes no more permissive than **0640** | no more permissive than **0600** |

The "must not exist" halves double as a guard: if a V2R9 change is ever back-ported here without its benchmark bump, verify goes red and says so.

## Why no Ansible pinning is needed (opposite of RHEL 8)

RHEL 9 / Rocky 9 ship Python 3.9 by default with full `dnf` Python bindings, so Ansible 2.19+ runs cleanly on managed nodes. RHEL 8's pinning constraint (platform-python 3.6 with dnf bindings vs. Python 3.9/3.11 without dnf bindings) does not apply here - use any current Ansible venv.

## Reference: what's in the default scenario

| File | Purpose |
|---|---|
| `molecule.yml` | Driver: docker. Image: `rockylinux/rockylinux:9-ubi-init`. The image is multi-arch and the platform is set from `${MOLECULE_DOCKER_PLATFORM:-linux/arm64}` - native on Apple Silicon by default, and CI exports `linux/amd64`. Host vars listed in the next table. |
| `prepare.yml` | Bootstraps minimal Rocky 9 UBI via `raw` (python3, sudo, iproute, openssh), then installs supporting packages (`audit`, `aide`, `crypto-policies`, `firewalld`, `chrony`, `rsyslog`, `logrotate`, `cronie`, `acl`, `kmod`, `dnf-plugins-core`, `python3-libselinux`, `python3-policycoreutils`, `python3-dnf-plugin-versionlock`, `tmux`, `git`, `procps-ng`, `openssl-pkcs11`, `opensc`). Stubs `/etc/default/grub` so GRUB-tag tasks don't fail in the container. |
| `converge.yml` | Sets root password (so PAM tasks don't lock us out), includes the role. |

## Reference: host vars (in `molecule/default/molecule.yml`)

| Override | Default | Purpose |
|---|---|---|
| `audit_output_destination` | `<working_dir>/_temp_fetched_audits/` (relative to `MOLECULE_PROJECT_DIRECTORY`) | Where fetched audit JSONs land. Pinned outside the role tree so they don't pollute system paths or git. |
| `rhel9stig_disruption_high` | `true` (molecule default) | Exercises high-disruption code paths (account locks, password lifetime resets, fapolicyd rules, sysctl reloads). Safe in throwaway containers; risky on real hosts. Defaults to `false` in `defaults/main.yml` for production safety. |
| `fetch_audit_output` | `true` | Auto-copies audit JSONs out of the container to `audit_output_destination`. |
| `system_is_container` | `true` | Gates container-incompatible tasks (auditd, FIPS, kernel sysctls, mount changes, GRUB, etc.) - see `vars/is_container.yml`. |
| `skip_reboot` | `true` | Suppresses reboot handlers (containers can't reboot). |

## Overriding the audit branch

The audit branch is **not** set in the molecule scenarios. It derives from `benchmark_version` in `defaults/main.yml`, which `vars/audit.yml` expands to `benchmark_{{ benchmark_version }}` and loads via `include_vars`. So bumping `benchmark_version` moves both scenarios in lock-step - there is no per-scenario pin to keep in sync (or to drift).

If you're QA'ing a feature branch on `RHEL9-STIG-Audit` that hasn't been merged yet, override at run time with extra-vars:

```bash
molecule converge -- --extra-vars 'audit_git_version=<your-qa-branch>'
```

`include_vars` (precedence 18) outranks molecule play/host vars, so a value set in `molecule.yml`/`converge.yml` would be ignored - `--extra-vars` (precedence 22) is the only way to override the audit branch at run time.

## Where to ask for help

- This role's open issues: `github.com/ansible-lockdown/RHEL9-STIG/issues`
- Ansible Lockdown Discord (linked from the role README badge)
- Molecule docs: `molecule.readthedocs.io`
- Goss output format: `github.com/krameff/goss`
