# RHEL9STIG

## Based on STIG V2R8 April 2026 - Lint Suppression Cleanup

- removed six dead `# noqa` directives that suppressed nothing. Five named `command-instead-of-module` (RHEL-09-214010 x2, RHEL-09-214030, RHEL-09-231110/231115/231120, RHEL-09-231200) and one named `shell-instead-of-command` (RHEL-09-411025). ansible-lint keys `command-instead-of-module` on the first word of the command, and the earlier shell/command pass prepended `set -o pipefail` to every shell task - so the first word became `set` rather than `mount` or `rpm`, the rule stopped matching, and the suppressions went silently dead. Verified by running ansible-lint with all six removed: still clean, zero `command-instead-of` findings
- the RHEL-09-411025 directive was never valid: `shell-instead-of-command` is not an ansible-lint rule (the real id is `command-instead-of-shell`), so it suppressed nothing from the day it was added. Even the correctly spelled rule would not fire there, because it exempts tasks that set `executable` and the command contains shell metacharacters
- three of the six sat inside the block scalar on the `rpm ...` line rather than on the module line, so they were part of the command string handed to the shell and ran as shell comments rather than as lint metadata
- the `command-instead-of-module` directive on the `Restart_auditd` handler is retained: its command begins with `service`, which is in the rule's module map, so that one genuinely suppresses a finding

## Based on STIG V2R8 April 2026 - August Public Issue Fixes

- templates/etc/aide.conf.j2: corrected the AIDE 0.18 version boundary. The input-database and verbosity gates tested `version_compare('0.18', '<=')` while the newer rule-syntax gates use `>=`, so at exactly AIDE 0.18.x the template rendered the legacy `database=` directive and `verbose=5` - both of which 0.18 removed - and `aide --init` failed. Both gates now use `<`, matching the comments already in the file and the `>=` gates further down. This is on the default path: `vars/main.yml` sets `aide_version_prechanges: '0.18'` as the fallback for `discovered_aide_version`, so any host where the aide package fact is unavailable rendered the broken config (addresses ansible-lockdown/RHEL9-STIG#182; thank you @pdlewisiw)
- tasks/prelim.yml: `PRELIM | Discover auditd rules files` searched the relative path `etc/audit/rules.d` instead of `/etc/audit/rules.d`. `ansible.builtin.find` does not fail on a missing path - it warns and returns an empty list - so RHEL-09-653110's file-mode sub-task looped zero times and the audit config modes were silently never applied (addresses ansible-lockdown/RHEL9-STIG#177; thank you @mbc3)
- handlers/main.yml: the `Rebuild_grub` handler now appends `--update-bls-cmdline` on RHEL 9.3 and later, matching DISA's own fixtext, which splits the rebuild command on that version boundary. Without the flag a bare `grub2-mkconfig` does not rewrite the BLS entries' `options=` lines, so the five controls that edit `GRUB_CMDLINE_LINUX` (RHEL-09-212035, 212040, 212050, 212055, 653120) never reached the running kernel cmdline on 9.3+. The flag is version-gated because it does not exist on 9.0-9.2 (addresses ansible-lockdown/RHEL9-STIG#176; thank you @mbc3)
- handlers/main.yml: the `Reload auditd rules` and `Schedule auditd reboot` handlers compared the registered result object `discovered_auditd_uid_immutable_check` against the strings `'0'` / `'1'` instead of its `.stdout`. A registered dict never equals a string, so that condition was always false and `augenrules --load` never ran after an audit rule change. The sibling condition on `discovered_auditd_unauth_immutable_check.stdout` was already correct
- handlers/main.yml: the auditd check, reload, reboot-schedule and restart handlers are now gated on `not system_is_container`. They previously ran in containers - reading `/etc/audit/audit.rules` and invoking `service auditd restart` - even though the auditd rules themselves are disabled there via vars/is_container.yml
- RHEL-09-215060: the TFTP secure-mode setting is now written as a systemd drop-in at `/etc/systemd/system/tftp.service.d/tftp.service.conf` (new template) instead of a `lineinfile` edit to the RPM-owned `/usr/lib/systemd/system/tftp.service`. Modifying the vendor unit changes its hash and makes RHEL-09-214030's `rpm -Va` integrity check report a finding - the same conflict RHEL-09-611195/611200 were changed to avoid. Only applies when `rhel9stig_is_tftp_server` is true

## Based on STIG V2R8 April 2026 - Repo QA CI

- added .github/workflows/repo_qa.yml: runs the Ansible-Lockdown QA checker as a pass/fail gate on pull requests to `main` and `devel`, pushes to `devel`, a weekly schedule and on demand. Roughly 90 seconds end to end, no secrets. Complements the Molecule workflow (which gates runtime behaviour) and runs in front of the tofu/EC2 pipelines rather than replacing them
- the checker is **checked out at pinned tag 2.8.1**, not pip-installed: several checks load helpers from the tool's scripts/ directory at runtime via importlib and that directory is not packaged (no `__init__.py`, and its pyproject declares no packages/py-modules), so a module-only install would silently degrade the audit_vars, shell_pipefail and rule_coverage checks. Pinning also means a checker release cannot change this repo's CI result on its own
- the workflow installs `yamllint==1.38.0` and `ansible-lint==26.4.0`, matching the versions pinned in .pre-commit-config.yaml so CI and the local hook agree. Both are optional dependencies of the checker guarded by `shutil.which`, so omitting them would leave those two checks reporting SKIP and quietly remove 2 of the checks capable of emitting FAIL
- the run uses `--no-report` (nothing is written into the repo; results go to the build log via `--console`) and `--skip spelling,grammar` (the two prose checks are excluded from the gate)
- the run uses `--strict`, which is load-bearing rather than cosmetic: most findings in this role are WARN severity, and a verified regression test showed an unquoted file mode exits 0 without `--strict` and 1 with it. Without the flag the gate would pass silently on that class of defect
- added .qa_baseline.json capturing the 8 pre-existing `Audit Variable Placement` findings (the 7 role-internal audit constants in defaults/main.yml plus the absent `audit_bin_validate_certs`; these are the V2R9 audit-variable alignment, which was never back-ported to V2R8). The baseline is keyed per finding on (file, description) with no line number, so unrelated line churn does not invalidate it, and a NEW finding in an already-baselined check still fails the build - verified by dropping one entry and confirming that finding alone resurfaced. Regenerate the baseline only deliberately, never to silence a fresh finding

## Based on STIG V2R8 April 2026 - Molecule CI

- added .github/workflows/molecule.yml: runs the `default` and `ubi` molecule scenarios on GitHub-hosted `ubuntu-latest` runners in a `fail-fast: false` matrix, on pull requests to `main` and `devel`, pushes to `devel`, a weekly schedule and on demand. A fast, secret-free gate in front of the existing tofu/EC2 pipelines, not a replacement for them. The workflow file is byte-identical to the one on the V2R9 branch
- the workflow runs `create` -> `converge` -> `verify` -> `destroy` rather than `molecule test`, deliberately omitting the `idempotence` step: converge run 2 for this role is a known non-zero-change run (the `rhel9stig_disruption_high`-gated RHEL-09-411090 pam_faillock/authselect settle, RHEL-09-252060 /etc/aliases, and the harness tasks that always report changed), so gating on it would fail for reasons that are not regressions. The goss audit JSONs upload as a `molecule-audit-<scenario>` artifact for inspection but do not gate the build, since post-scan failures are expected in a container. The gate is converge finishing `failed=0`, read alongside the goss deltas
- the container platform in molecule/default/molecule.yml and molecule/ubi/molecule.yml now reads from `${MOLECULE_DOCKER_PLATFORM:-linux/arm64}` instead of a hardcoded `linux/arm64`. Unset it resolves to the previous value, so local Apple Silicon runs are unchanged; CI exports `linux/amd64` for the amd64 runners. Both images are multi-arch. Note that `pre_build_image: true` means a cached image of the wrong architecture fails container creation rather than re-pulling, so forcing a non-native arch locally needs a `docker rmi` first
- the workflow installs `community.docker` and `ansible.posix` explicitly: the molecule docker driver requires both, `ansible-core` bundles no collections, and collections/requirements.yml does not carry `community.docker`. Python is pinned to 3.12 because the converge plays call `password_hash('sha512')`, which falls back to the stdlib `crypt` module removed in 3.13, and `passlib` is installed as it is declared nowhere in this repo
- added molecule/default/verify.yml and molecule/ubi/verify.yml (the ansible verifier was previously a no-op on this branch - it logged `Executed: Missing playbook` and exited 0, so both scenarios reported green having asserted nothing). ubi/verify.yml imports the default file so the assertions have a single source of truth, and the default file stats /etc/audit/rules.d/audit.rules before slurping so it tolerates UBI, where the `audit` and `aide` packages are subscription-gated and the file is never created
- **the V2R8 verify.yml is deliberately the inverse of the V2R9 branch's on three controls and must not be copied across branches wholesale**: RHEL-09-654215..654255 and RHEL-09-654097 assert the watch-style rules (`-w <path> -p wa -k <key>`) are PRESENT, where V2R9 asserts they are gone in favour of the syscall form; RHEL-09-211045 asserts the drop-in is at the suffixless /etc/systemd/system.conf.d/55-CtrlAltDel-BurstAction path, where V2R9 asserts the `.conf` suffix; RHEL-09-653110 asserts audit config modes no more permissive than 0640, where V2R9 tightened this to 0600. The paired "must not exist" assertions double as a guard - a V2R9 change back-ported here without its benchmark bump turns verify red and names itself
- molecule/README_Molecule_QuickStart.md documents the architecture override, the CI sequence and why idempotence is not gated, what verify.yml does and does not cover in a container, and the V2R8-vs-V2R9 assertion inversions

## Based on STIG V2R8 April 2026 - Benchmark V2R8 Upgrade

- RHEL-09-252035 (V-257948): added systemd-resolved support (addresses ansible-lockdown/RHEL9-STIG-Audit#31). When `rhel9stig_dns_processing_mode` is `systemd-resolved` the role now writes the configured nameservers to the `DNS=` key of `/etc/systemd/resolved.conf` (`[Resolve]` section, via ini_file) and notifies a new `Restart_systemd-resolved` handler - under systemd-resolved `/etc/resolv.conf` holds only the 127.0.0.53 stub, so the servers must be set in resolved.conf. Also exposed `rhel9stig_dns_processing_mode` through the goss bridge template so the paired audit selects the correct file to check
- renamed the goss bridge template `templates/ansible_vars_goss.yml.j2` -> `templates/lockdown_audit.yml.j2` to match the V2R9 New Alignment Strategy naming, and updated the `src:` reference in tasks/pre_remediation_audit.yml (rendered dest and `audit_vars_path` unchanged)
- V2R8 benchmark bump (446 -> 446 rules; updates-only, no add/remove)
- 8 rules severity-bumped medium -> high: RHEL-09-215100, 215105, 255064, 255065, 255070, 255075, 671020, 672050 (moved Cat2 -> Cat1 across grouped task files; tags updated to CAT1)
- 27 SV-* tag revisions updated to V2R8 IDs (context-aware replacement anchored on RHEL-09-NNNNNN)
- 17 PRE-EXISTING SV-ID drifts in tasks fixed to V2R8 XCCDF values (V2R7 cycle missed these); 3 had wrong base SV numbers - V-* tags corrected for RHEL-09-231120 (V-257863 -> V-257865), RHEL-09-253025 (V-257959 -> V-257960), RHEL-09-411065 (V-258051 -> V-258052)
- benchmark_version bumped v2r7 -> v2r8 in defaults/main.yml
- README banner updated V2R7 -> V2R8 with new DISA download URL
- tasks/Cat2/RHEL-09-672xxx.yml removed (only rule 672050 lived there; emptied by sev-bump move) and corresponding import removed from tasks/Cat2/main.yml
- molecule audit_git_version pinned to benchmark_v2r8 in default scenario
- molecule hygiene (post-release fix): removed the inert `audit_git_version` pins from both scenarios - the ubi scenario had drifted to `benchmark_v2r7` while default was `benchmark_v2r8`, but neither actually took effect because `vars/audit.yml` loads `audit_git_version` via `include_vars` (precedence 18, which outranks molecule host/play vars) and derives it as `benchmark_{{ benchmark_version }}`. Both scenarios now track the single `benchmark_version` source; QuickStart README updated to match; override a QA branch at run time with `--extra-vars 'audit_git_version=<branch>'`
- fixed templates/ansible_vars_goss.yml.j2 duplicate top-level rhel9stig_dns_servers key: IPv4 and IPv6 blocks merged into a single emit (dual-stack render previously produced invalid YAML)
- removed orphan `listen: Restart_sysctl` alias on Reload_sysctl handler (dead code - all 17 notify refs target the handler name directly)
- RHEL-09-611180: enable `pcscd.socket` instead of the `pcscd` service, per the V2R8 XCCDF fixtext (`systemctl enable --now pcscd.socket`); added the missing CCI-004046 tag. Aligns with the RHEL9-STIG-Audit socket check. Task title left as the DISA-verbatim "pcscd service".
- RHEL-09-431015: corrected the task title to the DISA-verbatim "RHEL 9 must enable the SELinux targeted policy" (was a stale faillock-context description; logic and IDs were already correct)
- RHEL-09-212055: corrected the four sub-task titles that were copy-pasted from 212050 ("mitigations against processor-based vulnerabilities") to "enable auditing of processes that start prior to the audit daemon" (logic already correct)
- trimmed trailing blank line at end of tasks/Cat2/RHEL-09-215xxx.yml (yamllint empty-lines error)
- molecule default scenario aligned with RHEL8-STIG sibling: enabled rhel9stig_disruption_high (exercises full code path in throwaway container), fetch_audit_output (auto-copies audit summary out of container), audit_output_destination pinned to <working_dir>/_temp_fetched_audits/; prepare.yml adds openssl-pkcs11 + opensc for PAM smartcard coverage
- added `rhel9stig_shell_executable` (default `/bin/bash`) in vars/main.yml and applied `args: executable` to the `ansible.builtin.shell` tasks that require a shell (pipe, glob, `&&`, or a shell builtin) so the interpreter is explicit and overridable to a POSIX shell such as dash or ash for container runs
- converted single-command shell tasks that need no shell to `ansible.builtin.command` (RHEL-09-212010 grub-password check, RHEL-09-232190/232195/232205/232215, RHEL-09-611010 system-auth/password-auth checks, and the prelim sudoers-file find)
- converted `ansible.builtin.command` tasks that actually use shell features to `ansible.builtin.shell` so they work as intended (the command module was passing `|`/`>`/`&&`/globs as literal arguments): RHEL-09-215035, RHEL-09-232260, RHEL-09-411045, RHEL-09-652040, RHEL-09-652045
- standardized every remaining `ansible.builtin.shell` task on the multi-line block-scalar (`|`) form with `set -o pipefail` as the first line, so the convention is uniform across the role; non-piped shell tasks (glob, `&&`, redirect) now carry `set -o pipefail` too, where it is a harmless no-op
- RHEL-09-652050: rewrote the grep alternation `'a|b'` as `grep -e 'a' -e 'b'` (identical match) so the `|` is unmistakably part of the regex, not a shell pipe
- added molecule/README_Molecule_QuickStart.md - quick-start guide for the default scenario covering venv (no 2.16 pinning needed on RHEL 9), host_vars, expected results, and gating-run command sequence
- fixed RHEL-09-252050 PATCH when-clause: removed literal-string quoting around the `or rhel9stig_postfix_client_conf not in discovered_postfix_client_restrict.stdout` expression (Ansible 2.19+ rejects string-as-bool; only surfaces on a 2nd converge where postfix already has client_restrictions set, masking on initial-apply via length==0 short-circuit)
- fixed README Technical Dependencies section: Python3.8 -> Python 3.9+, Ansible 2.12+ -> Ansible 2.16+, python-def typo -> python3-dnf, libselinux-python -> python3-libselinux (matches meta/main.yml min_ansible_version 2.16.1 and current RHEL 9 default Python)
- fixed templates/etc/aide.conf.j2 for aide >= 0.18 compatibility (addresses public issue #161): renamed `database=` to `database_in=` (input directive renamed in aide 0.18); removed `verbose=5` directive (replaced by log_level/report_level in aide 0.18). `file:` URL prefix retained on database_in / database_out / report_url (plain paths are rejected by aide >= 0.18). RHEL-09-651020 / RHEL-09-651025 templates affected; previously caused `aide --init` rc=17 with "unexpected character: ':'" on RHEL 9.8 / aide-0.19.2 hosts.
- made templates/etc/aide.conf.j2 version-gated to support both AIDE < 0.18 (legacy `database=`, `verbose=5`) and AIDE >= 0.18 (`database_in=`, no verbose). Adds a post-install `package_facts` re-gather + `discovered_aide_version` set_fact in the RHEL-09-651010 block (`tasks/Cat2/RHEL-09-651xxx.yml`); `vars/main.yml` gets a safe `discovered_aide_version: '0.18'` default so the template renders correctly even when 651010 is disabled. `file:` URL prefix retained in both branches. Restores RHEL 9.0-9.3 (aide 0.16.x) host coverage while keeping the aide-0.18+ fix from the previous entry.
- Thank you @hectoralicea for submitting issue ansible-lockdown/RHEL9-STIG#161
- Thank you @uk-bolly for the review

- templates/etc/aide.conf.j2 also drops the upcoming `+S` deprecation warning by emitting `+growing` for AIDE >= 0.18 in the ALL and FIPSR rule definitions; renamed the `CONTENT_EX` rule alias to `EXTCONTENT` throughout the template (no functional change; both identifiers are valid AIDE variable names).

- migrated remaining bare `ansible_*` variable references to `ansible_facts['name']` form across defaults/main.yml, tasks/Cat1/RHEL-09-6xxxxx.yml, tasks/Cat2/RHEL-09-251xxx.yml, tasks/Cat2/RHEL-09-252xxx.yml, tasks/main.yml, tasks/prelim.yml, and vars/main.yml (ahead of upcoming Ansible deprecation of the flat fact namespace).

- fixed tasks/main.yml connecting-user password assert: the register var captured by the shell task and the var referenced in the assert task now agree (`discovered_ansible_user_password_set`); previously the assert referenced a different register name, so the assert was effectively unreachable.
- renamed 2 further AUDIT-named sub-tasks that modified state to PATCH (read-only contract): RHEL-09-232260 "/ scan" -> "/ relabel" (`restorecon -v`, changed_when:true) and RHEL-09-611160 "get state" -> "set driver" (`opensc-tool --set-conf-entry`, changed_when:true). Both invoked state-mutating commands under an `| AUDIT |` name, so `--tags AUDIT` / `--check` runs silently relabeled SELinux contexts or rewrote the opensc card-driver config. The read-only first sub-task of each control (find scan / get-conf-entry, changed_when:false) remains AUDIT.
- QA hygiene sweep (pre-existing defects surfaced by full QA, not from the V2R7 carry-forward): fixed RHEL-09-271100 toggle typo `mrhel_09_271100` -> `rhel_09_271100` (the control was unmanageable and referenced an undefined var in `when:`); fixed RHEL-09-231030 WARN sub-task name mislabeled as RHEL-09-231025 (tasks/Cat3/RHEL-09-2xxxxx.yml); restored RHEL-09-411060 identifiers `SV-258051r991589_rule` + `V-258051` on the consolidated 411060/411065 task (the V2R8 bump had wrongly duplicated 411065's `SV-258052`/`V-258052` in their place, dropping 411060's own SV/V tags); removed unused `Remount_var_log` handler (no /var/log mount-option control notifies it); fixed `templates/etc/resolv.conf.j2` optional-DNS path (loop var `{{ domains }}`, interpolated `options {{ rhel9stig_resolv_dns_options }}`); README typo fixes (compliant, has, controls/needs); aligned molecule QuickStart README to public-safe RHEL9-STIG naming.

- updated the audit binary source in `vars/audit.yml` to the krameff goss fork and bumped the downloaded goss release `v0.4.8` -> `v0.5.0` (with the krameff v0.5.0 sha256 checksums), matching the paired RHEL9-STIG-Audit and the sibling Lockdown roles (RHEL9-CIS, RHEL10-STIG); the previous `goss-org/goss` v0.4.8 download no longer matched the audit content
- Corrected the goss binary link in `README.md` and `molecule/README_Molecule_QuickStart.md` from `goss-org/goss` to the krameff fork (`krameff/goss`), matching the migrated audit binary source (the prose links were missed when the goss source moved to krameff)

Public community issue fixes (July 2026):
- RHEL-09-411080 / 411085 / 411090 / 412045 / 611030: fixed the `Authselect_enable_faillock` handler, which used `ansible.builtin.command` with a shell pipe (`authselect current | grep failock`) - the pipe was passed as a literal argument so the grep never ran, and `failock` was misspelled. Rewrote it as `ansible.builtin.shell` with `set -o pipefail` and the correct `faillock` match so the authselect faillock feature state is detected reliably before enabling it (addresses #166; thank you @seanlongcc).
- RHEL-09-431016 / 432015 / 432020 / 432025 / 432030: the sudoers-file loops referenced `prelim_sudoers_files.files` and `item.path`, but the PRELIM task registers an `ansible.builtin.command` result (which exposes `.stdout_lines`, not `.files`), so every consuming task silently skipped. Switched the consumers to `prelim_sudoers_files.stdout_lines` and `item`, and added `rhel_09_431016` to the PRELIM task run condition so the list is populated for 431016 as well (addresses #168; thank you @mbc3).
- RHEL-09-212020: the control set `set superusers=` in `/etc/grub.d/01_users` but left the `password_pbkdf2 root ${GRUB2_PASSWORD}` line bound to `root`, so a non-root `rhel9stig_grub_superuser` ended up with no bootloader password. Added a `replace` that aligns the `password_pbkdf2` user to `rhel9stig_grub_superuser` (guarded on `rhel9stig_set_bootloader_password`) (addresses #170; thank you @mbc3).
- RHEL-09-611195 / 611200: stopped commenting out `ExecStart`/`ExecStartPre` in the RPM-owned base units `/usr/lib/systemd/system/{emergency,rescue}.service`, which made `RHEL-09-214030` (`rpm -Va` integrity) report those files. The controls now deploy a proper systemd override drop-in (templated `emergency.service.conf` / `rescue.service.conf` with an `ExecStart=` reset followed by the sulogin `ExecStart`) under `/etc/systemd/system/*.service.d/`, leaving the vendor unit files untouched - satisfying both the 611195/611200 check and 214030 (addresses #172; thank you @mbc3).
- RHEL-09-252035: the `/etc/resolv.conf` template sub-task fired whenever `discovered_dns_nm_set` was defined (which it almost always was once 252040 had run), overwriting a systemd-resolved-managed `resolv.conf` and breaking its symlink. Restricted it to NetworkManager DNS modes `none`/`unmanaged` so systemd-resolved-managed hosts are left alone (addresses #173; thank you @mbc3).

## Based on STIG V2R7 - 2026 May QA Final updates

- Renamed 6 AUDIT-named tasks that modified state to PATCH (read-only contract). Tasks named `| AUDIT |` must not invoke modifying modules; `--tags AUDIT` and `--check` runs were silently mutating state. Affected: RHEL-09-215105 "Add required pmod files" (template), RHEL-09-232045 "update permissions" (file with mode/owner), RHEL-09-251030 (lineinfile to /etc/firewalld/firewalld.conf), RHEL-09-411090 "no authselect" both password-auth and system-auth variants (lineinfile), RHEL-09-412035 "create file if absent" + "Edit file if present" (template + lineinfile). The 411090 password-auth task name suffix also disambiguated to "password-auth no authselect" so it no longer collides with the system-auth sibling.
- `set -o pipefail` added to 25 single-line `ansible.builtin.shell:` tasks with piped commands across `tasks/prelim.yml` (10) and `tasks/Cat2/RHEL-09-{212,213,214,232}xxx.yml` (15). All converted to the multi-line block-scalar shell form matching the existing convention in `tasks/parse_etc_passwd.yml`, `tasks/pre_remediation_audit.yml`, and `tasks/post_remediation_audit.yml`. Without pipefail, early-pipeline failures (e.g. grep rc=2 from a missing file) were masked by the final command's rc=0 and the failed_when conditions never tripped, allowing silently corrupt registered variables.
- PRELIM NetworkManager DNS state task: `failed_when:` accepts rc=2 (file absent) in addition to rc=0/1. Required follow-up to the `set -o pipefail` change because grep's rc=2 (NetworkManager.conf missing on minimal containers) now propagates through the pipeline instead of being masked by sed's rc=0. On systems without NetworkManager.conf the task now succeeds with an empty stdout, matching the pre-pipefail behavior.
- `no_log: true` added to 4 tasks that handle shadow-file content or lock accounts: RHEL-09-411015 AUDIT (awk on /etc/shadow for pass-max-days), RHEL-09-611080 AUDIT (awk on /etc/shadow for 24-hour restriction), RHEL-09-671015 AUDIT (cat/grep on /etc/shadow for non-FIPS hashes), RHEL-09-611155 PATCH (`ansible.builtin.user password_lock` looping empty-password accounts). Prevents password hash leakage in verbose Ansible output and Ansible logs.
- Lint
- Alignment
- dup control removed
- /var check fixed - typo
- ordering updated
- fixed conditionals and dconf logic
- removed committed .DS_Store artifact
- container detection now covers community.docker.docker connection plugin
- CONTRIBUTING.md branding aligned to Ansible-Lockdown (hyphen)
- README Twitter URL migrated to x.com
- Rocky vars now define rhel9stig_rule_enable_repogpg override (parity with RedHat/AlmaLinux/OracleLinux)
- typo fix in is_container.yml comment for rhel_09_232255 (rrhel9stig_ -> rhel9stig_)
- ansible_vars_goss template now outputs rhel_09_271095 toggle (audit test was using zero/empty value)
- register: moved after failed_when: in 232xxx and 271xxx bundle tasks (per Lockdown convention)
- absolute file modes converted to relative notation in tasks/main.yml and 232xxx
- RHEL-09-433016 fapolicy audit failed_when tolerates rc=127 when fapolicyd-cli is absent
- Firewalld_reload and Restart_NetworkManager handlers now skip in containers
- molecule default scenario added for local QA testing (Rocky 9 docker)
- galaxy.yml removed (not required for private repo)
- is_container.yml cleaned up - removed 4 orphan TMUX entries (412010/412015/412025/412030 no longer in defaults)
- is_container.yml inline per-control comments stripped for consistency
- is_container.yml controls grouped by STIG ID prefix with section headers (211xxx, 212xxx, etc.)
- molecule scenarios: remove yaml stdout_callback (incompatible with ansible-core 2.19 + community.general 9.4.0)
- molecule ubi scenario added for redhat/ubi9 cross-image testing alongside the default rockylinux9 scenario
- molecule images switched to multi-arch ubi-init variants (rockylinux/rockylinux:9-ubi-init, redhat/ubi9-init:latest) - native arm64 on Apple Silicon, systemd as PID 1 without a Dockerfile
- molecule ubi prepare stubs /etc/audit, /etc/audit/rules.d, /etc/aide, /etc/aide/aide.conf.d since those packages are subscription-gated in public UBI repos
- defaults/main.yml documentation improved - added WARNING block on variable precedence, richer comments for setup_audit/run_audit/get_audit_binary_method/audit_content/disruption_high, exposed change_requires_reboot
- defaults/main.yml list vars indented with 2-space sequence style for consistency
- devel_pipeline_validation.yml IAC_BRANCH if-else block normalized to 2-space indent (matches main_pipeline_validation.yml)
- export_badges_private.yml dead conditional removed (referenced github.event_name == 'schedule' but no schedule trigger is defined)
- tasks/main.yml connecting-user check: fixed undefined variable rhel10stig_playbook_user -> rhel9stig_playbook_user and stray trailing quote in task name
- rsyslog remote-server var aligned end-to-end (defaults rhel9stig_rsyslog_remote_server_ip; legacy lineinfile, rainerscript template, and audit bridge updated to match)
- tasks/prelim.yml authselect prelim task: replaced undefined dict ref rhel9stig_authselect['custom_profile_name'] with scalar rhel9stig_authselect_custom_profile; corrected duplicated STIG IDs in name (411080|411080|411090 -> 411080|411085|411090) and when condition (411085 or 411085 or 411090 -> 411080 or 411085 or 411090)
- tasks/Cat3/*.yml: replaced CAT2 tag with CAT3 on all 15 Cat3 controls (selective `--tags CAT3` runs were silently empty)
- tasks/Cat2/RHEL-09-653xxx.yml: 653025 task tag corrected from RHEL-09-653055 to RHEL-09-653025
- tasks/Cat2/RHEL-09-654xxx.yml: 654245 shadow-audit task now uses rhel_09_654245 toggle and RHEL-09-654245 tag (previously both pointed at 654240)
- Task name titles aligned verbatim to V2R7 XCCDF for 12 controls where the role title was inverted, cross-pasted from an adjacent rule, contained discussion text, or otherwise diverged: 213090 (storage->disable storing), 231170 (noexec->nosuid), 252015 (chrony package->chronyd service), 271080 (idle-delay->lock-delay), 431020 (SELinux targeted policy->faillock tally directory context), 611180 (pcsc-lite package->pcscd service), 651020/651025 (file integrity tool->cryptographic mechanisms audit tools), 214025 (locally installed packages->all software repositories), 251035 (discussion text->PPSM CAL rule title), 291010 (discussion text->disable USB mass storage), 411015 (added scope qualifiers removed)
- RHEL-09-214025 find repo files task: removed use_regex:true (incompatible with glob pattern *.repo); the find module was silently returning zero files due to "nothing to repeat at position 0" regex error, leaving the gpgcheck=1 replace loop with an empty list. CAT-1 control now executes correctly.
- RHEL-09-214025 Set gpgcheck task: path: "{{ item }}" -> path: "{{ item.path }}" (find returns stat dicts, not path strings); regexp tightened from ^gpgcheck (which matched only the word "gpgcheck" and left "gpgcheck=0" partially replaced as "gpgcheck=1=0") to ^gpgcheck\s*=.*$ to replace the full key=value line per the XCCDF fix-text. Surfaced by molecule failure once the find no-op was fixed.
- RHEL-09-611195/611200 copy service file: added force:false to the copy task that clobbered the lineinfile-edited drop-in on every converge. Goss audit was flipping these two controls from pass to fail between converges because the copy ran before the lineinfile re-edit. Audit state now stable run-to-run.
- RHEL-09-271065 ini_file: added no_extra_spaces:true so the dconf drop-in writes idle-delay=uint32 600 (no spaces around =) per dconf format convention, matching the audit goss content regex.
- RHEL-09-231085 when-toggle alignment: parent block when: was rhel_09_231080 (gated by sibling control's toggle); corrected to rhel_09_231085 so the control honors its own toggle.
- RHEL-09-653015 task name: added trailing period to align verbatim with V2R7 XCCDF title.
- RHEL-09-432035 task name: changed outer YAML quoting from double to single to preserve XCCDF's literal "su" double-quoted command name (was 'su' single-quoted).
- RHEL-09-271105 AUDIT sub-task: changed gsettings set -> gsettings get (the discovery task was destructively writing the value before register could capture state).

## Based on STIG V2R7 - 05 Jan 2026 - May 26 update for public release

Public issue #154 addressed thanks to @PrymalInstynct
Public issue # 157 and #158 addressed thanks to @hectoralicea
names updated to show PATCH/AUDIT where needed.
octal mode change dto symbolic
typo updates
tidy up var naming convention - prelim, discovered and unique naming
remove unused variables
connecting user test updated
vars moved to task rather than on blocks

### Initial

Linting
company name alignment
ssh variables separated so easier to override
multiple control updated to iprove remediation

RuleID updates to all plus comments

Cat I
- 211010 - RuleID updated - updated supported OS version minimum
- 211045 - Rewritten to use drop in file replacement
- 212020 - RuleID updated
- 214025 - RuleID updated
- 215060 - RuleID updated and control with conditional
- 671010 - updated to add Grubby fips enabled
- 672020 - RuleID updated

Cat II

- 213010 - moved to drop in file
- 213015 - moved to drop in file
- 213020 - moved to drop in file
- 213025 - moved to drop in file
- 213030 - moved to drop in file
- 213035 - moved to drop in file
- 213070 - moved to drop in file
- 213075 - moved to drop in file
- 213080 - moved to drop in file
- 213085 - wont run if 213040 is true
- 213090 - wont run if 213040 is true
- 213095 - wont run if 213040 is true
- 213100 - wont run if 213040 is true
- 213105 - moved to drop in file
- 214030
- 215035 - no longer rsh-server now disable epel - new variable option
- 215045
- 215101
- 231105
- 231110
- 231115
- 231120
- 231200 - excluded vfat filesystem mount from search
- 232040 - Added group and change mode to symbolic
- 232240 - changed to capture any system user greater than UID 999
- 252040 - Extended variable option for NetworkManager DNS options default none
- 251035 - moved to drop in file
- 253010 - moved to drop in file
- 253015 - moved to drop in file
- 253020 - moved to drop in file
- 253025 - moved to drop in file
- 253030 - moved to drop in file
- 253040 - moved to drop in file
- 253045 - moved to drop in file
- 253050 - moved to drop in file
- 253055 - moved to drop in file
- 253060 - moved to drop in file
- 253065 - moved to drop in file
- 253070 - moved to drop in file
- 253075 - moved to drop in file
- 254010 - moved to drop in file
- 254105 - moved to drop in file
- 254020 - moved to drop in file
- 254025 - moved to drop in file
- 254030 - moved to drop in file
- 254035 - moved to drop in file
- 254040 - moved to drop in file
- 255100
- 255115 - added owner and group
- 255130
- 271065 - updated value to 10mins from 15min
- 411115 - Removed
- 412075 - Removed
- 412080 - changed timeout to 10mins
- 431016
- 432025
- 611160
- 611170
- 611195 - rewrite - drop in file
- 611200 - rewrite - drop in file
- 611190 - rewritten to have check no longer manual
- 652025 - Improved test
- 652055 - rewritten and rainer script option added
- 653040
- 653090
- 653110
- 654101
- 654015
- 654020
- 654025
- 654065
- 654070
- 654075
- 654080
- 654096 - now 654097
- 654205
- 654210
- 654260 - removed

## Based on STIG V2R5 07 August 2025 - Feb26 updates
- 611160 updated
- 232190 updated
- 232195 updated
- company title update
- audit improvements


## 2.5.0 Based on STIG V2R5 07 August 2025

- added extra options and explanation for audit component
- updated aide checks
- fixed v2.19 compliance and conditionals

- Removed Requirements
  - RHEL-09-255025
  - RHEL-09-255055
  - RHEL-09-255060
  - RHEL-09-653115
  - RHEL-09-672025

- Added Requirements
  - RHEL-09-654096 - New rule to audit

- RuleID update for all listed
  - RHEL-09-212020
  - RHEL-09-213010
  - RHEL-09-213015
  - RHEL-09-213020
  - RHEL-09-213025
  - RHEL-09-213030
  - RHEL-09-213035
  - RHEL-09-213040
  - RHEL-09-213070
  - RHEL-09-213075
  - RHEL-09-213080
  - RHEL-09-213105
  - RHEL-09-251045
  - RHEL-09-253010
  - RHEL-09-253015
  - RHEL-09-253020
  - RHEL-09-253025
  - RHEL-09-253030
  - RHEL-09-253035
  - RHEL-09-253040
  - RHEL-09-253045
  - RHEL-09-253050
  - RHEL-09-253055
  - RHEL-09-253060
  - RHEL-09-253065
  - RHEL-09-253075
  - RHEL-09-254010
  - RHEL-09-254015
  - RHEL-09-254020
  - RHEL-09-254025
  - RHEL-09-254030
  - RHEL-09-254035
  - RHEL-09-254040
  - RHEL-09-215015
  - RHEL-09-215060
  - RHEL-09-215105
  - RHEL-09-231115
  - RHEL-09-232020
  - RHEL-09-232180
  - RHEL-09-232185
  - RHEL-09-232200
  - RHEL-09-232205
  - RHEL-09-251020
  - RHEL-09-251035
  - RHEL-09-252065
  - RHEL-09-432025
  - RHEL-09-432030
  - RHEL-09-611085
  - RHEL-09-611160
  - RHEL-09-611200
  - RHEL-09-651010
  - RHEL-09-651025
  - RHEL-09-652010
  - RHEL-09-652055
  - RHEL-09-653035
  - RHEL-09-653090
  - RHEL-09-653120
  - RHEL-09-654010
  - RHEL-09-654015
  - RHEL-09-654020
  - RHEL-09-654025
  - RHEL-09-654065
  - RHEL-09-654070
  - RHEL-09-654075
  - RHEL-09-654080
  - RHEL-09-654205
  - RHEL-09-654210
  - RHEL-09-654096
  - RHEL-09-654220
  - RHEL-09-672020

## 2.4.0 Based on STIG V2R4 02 April 2025

- 2025_October_Updates
  - Addresses issue #106, Thank you @ccravens
  - Addresses issue #116, Thank you @wdower
  - Addresses issue #127, Thank you @dsexton18
  - Addresses issue #128, Thank you @padili-metrostar

- RuleID update for all listed
  - RHEL-09-212020 - ID
  - RHEL-09-212045 - title and requirements
  - RHEL-09-215060 - ID
  - RHEL-09-215101 - New control to install postfix
  - RHEL-09-232040 - ID and updated
  - RHEL-09-232200 - check files only
  - RHEL-09-232205 - ID
  - RHEL-09-255045 - ID
  - RHEL-09-255105 - All config files
  - RHEL-09-255110 - All config files
  - RHEL-09-255115 - title and requirement
  - RHEL-09-411045 - ID
  - RHEL-09-431016 - New control selinux
  - RHEL-09-611205 - rule removed
  - RHEL-09-232265 - rule removed
  - RHEL-09-654025 - ID updated
  - RHEL-09-671095 - ID updated

## 2.3.0 Based on STIG V2R3 Jan28 2025

- RuleID Updates
- CCI Updates
- package removals/additions now don't skip if package present or not but run through giving correct state
- Upgraded several control to disruption_high
- authselect updates for faillock related controls
- var dictionaries renamed to allow easier overriding of variables
- RHEL-09-672010 - Becomes RHEL-09-215100
- RHEL-09-672020 - moved to CAT1 and approach changed to remediate inline with documentation
- RHEL-09-672030 - removed
- RHEL-09-171011 - Added
- RHEL-09-232103 - Added
- RHEL-09-232104 - Added
- RHEL-09-251025 - Added firewalld reload as issues seen for new connections
- RHEL-09-255064 - Added
- RHEL-09-255070 - Added
- RHEL-09-433016 - Added fapolicyd aiding rules and testing - new var rhel9stig_allow_fapolicy_updates
- RHEL-09-610205 - title update
- RHEL-09-652035 - removed
- RHEL-09-653110 - added audit.rules
- RHEL-09-653130 - moved to 653xxx as 652035 no longer present
- RHEL-09-672035 - removed
- RHEL-09-672040 - removed
- RHEL-09-672045 - moved to 215105

## 2.2.0 Based on STIG V2R2 Oct24 2024

- RuleID updates
- NIST ID updates
- tmux no longer required removed controls:
  - RHEL-09-412010
  - RHEL-09-412015
  - RHEL-09-412020
  - RHEL-09-412025
  - RHEL-09-412030
- RHEL-09-611085 - enhance with sudoers nopasswd exclude list
- RHEL-09-412035 - Changed tmout to be consistent across STIGS.
- lint files updated
- new lint layout
- file mode changed to symbolic for greater idempotency
- Aide logic rewritten
- nested variables removed and renamed
  - aide
  - auditd

Many rules now linked with nist and CCI (not on official revision history)

## 2.1.0 Based on STIG V2R1 Jul24 2024

- Every control ruleid updates due to STiG new CMS
- Removed as no longer required
  - RHEL-09-211025
  - RHEL-09-611016
  - RHEL-09-611020
- Following updated NIST relationships
  - RHEL-09-653010
  - RHEL-09-213020
  - RHEL-09-214010
  - RHEL-09-214015
  - RHEL-09-214020
  - RHEL-09-214025
  - RHEL-09-215010
  - RHEL-09-215075
  - RHEL-09-653015
  - RHEL-09-654215
  - RHEL-09-654220
  - RHEL-09-654225
  - RHEL-09-654230
  - RHEL-09-654235
  - RHEL-09-654240
  - RHEL-09-252010
  - RHEL-09-252015
  - RHEL-09-252020
  - RHEL-09-255035
  - RHEL-09-255045
  - RHEL-09-255100
  - RHEL-09-271045
  - RHEL-09-271050
  - RHEL-09-271055
  - RHEL-09-271060
  - RHEL-09-654245
  - RHEL-09-411010
  - RHEL-09-411015
  - RHEL-09-411050
  - RHEL-09-412010
  - RHEL-09-432015
  - RHEL-09-432025
  - RHEL-09-432035
  - RHEL-09-611010
  - RHEL-09-611040
  - RHEL-09-611050
  - RHEL-09-611055
  - RHEL-09-611060
  - RHEL-09-611065
  - RHEL-09-611070
  - RHEL-09-611075
  - RHEL-09-611080
  - RHEL-09-611085
  - RHEL-09-611090
  - RHEL-09-611095
  - RHEL-09-611100
  - RHEL-09-611110
  - RHEL-09-611115
  - RHEL-09-611120
  - RHEL-09-611125
  - RHEL-09-611130
  - RHEL-09-611135
  - RHEL-09-611140
  - RHEL-09-611145
  - RHEL-09-611150
  - RHEL-09-611160
  - RHEL-09-611165
  - RHEL-09-611170
  - RHEL-09-611175
  - RHEL-09-611180
  - RHEL-09-611185
  - RHEL-09-631010
  - RHEL-09-671015
  - RHEL-09-671025
  - RHEL-09-291010
  - RHEL-09-291015
  - RHEL-09-291020

## 1.3.0 Based on STIG V1r3 Jan24 2024

- RuleIDs updated
  - RHEL-09-212045
  - RHEL-09-213060
  - RHEL-09-215060
  - RHEL-09-255025
  - RHEL-09-255030
  - RHEL-09-255035
  - RHEL-09-255040
  - RHEL-09-255045
  - RHEL-09-255050
  - RHEL-09-255055
  - RHEL-09-255080
  - RHEL-09-255085
  - RHEL-09-255090
  - RHEL-09-255095
  - RHEL-09-255100
  - RHEL-09-255130
  - RHEL-09-255135
  - RHEL-09-255140
  - RHEL-09-255145
  - RHEL-09-255150
  - RHEL-09-255155
  - RHEL-09-155160
  - RHEL-09-255165
  - RHEL-09-255170
  - RHEL-09-255175

- RHEL-09-255070 removed as duplicate of 255075
  - RHEL-09-255075 updated

## 1.2.1 Based on STIG V1R2 Jan24 2024

- precommit updates
- issues
  - #12 thanks to @layluke
  - #13 thanks to @PoundsOfFlesh - some excellent items from PR
  - update audit summary output

## 1.2 Based on STIG V1R2 Jan24 2024

- control updates
- pre-commit updates
- rule IDs
- lint
- audit updates
- tag updates
- issues
  - #2
  - #3
  - #4

## 1.1 Based on STIG V1R1

Initial release
