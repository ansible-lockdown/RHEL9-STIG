# RHEL9STIG

## Based on STIG V2R7 - 2026 May QA Final updates

Lint
Alignment
dup control removed
/var check fixed - typo
ordering updated
fixed conditionals and dconf logic
removed committed .DS_Store artifact
container detection now covers community.docker.docker connection plugin
CONTRIBUTING.md branding aligned to Ansible-Lockdown (hyphen)
README Twitter URL migrated to x.com
Rocky vars now define rhel9stig_rule_enable_repogpg override (parity with RedHat/AlmaLinux/OracleLinux)
typo fix in is_container.yml comment for rhel_09_232255 (rrhel9stig_ -> rhel9stig_)
ansible_vars_goss template now outputs rhel_09_271095 toggle (audit test was using zero/empty value)
register: moved after failed_when: in 232xxx and 271xxx bundle tasks (per Lockdown convention)
absolute file modes converted to relative notation in tasks/main.yml and 232xxx
RHEL-09-433016 fapolicy audit failed_when tolerates rc=127 when fapolicyd-cli is absent
Firewalld_reload and Restart_NetworkManager handlers now skip in containers
molecule default scenario added for local QA testing (Rocky 9 docker)
is_container.yml cleaned up - removed 4 orphan TMUX entries (412010/412015/412025/412030 no longer in defaults)
is_container.yml inline per-control comments stripped for consistency
is_container.yml controls grouped by STIG ID prefix with section headers (211xxx, 212xxx, etc.)
molecule scenarios: remove yaml stdout_callback (incompatible with ansible-core 2.19 + community.general 9.4.0)
molecule ubi scenario added for redhat/ubi9 cross-image testing alongside the default rockylinux9 scenario
molecule images switched to multi-arch ubi-init variants (rockylinux/rockylinux:9-ubi-init, redhat/ubi9-init:latest) - native arm64 on Apple Silicon, systemd as PID 1 without a Dockerfile
molecule ubi prepare stubs /etc/audit, /etc/audit/rules.d, /etc/aide, /etc/aide/aide.conf.d since those packages are subscription-gated in public UBI repos
defaults/main.yml documentation improved - added WARNING block on variable precedence, richer comments for setup_audit/run_audit/get_audit_binary_method/audit_content/disruption_high, exposed change_requires_reboot
defaults/main.yml list vars indented with 2-space sequence style for consistency
devel_pipeline_validation.yml IAC_BRANCH if-else block normalized to 2-space indent (matches main_pipeline_validation.yml)
export_badges_private.yml dead conditional removed (referenced github.event_name == 'schedule' but no schedule trigger is defined)
tasks/main.yml connecting-user check: fixed undefined variable rhel10stig_playbook_user -> rhel9stig_playbook_user and stray trailing quote in task name
rsyslog remote-server var aligned end-to-end (defaults rhel9stig_rsyslog_remote_server_ip; legacy lineinfile, rainerscript template, and audit bridge updated to match)
tasks/prelim.yml authselect prelim task: replaced undefined dict ref rhel9stig_authselect['custom_profile_name'] with scalar rhel9stig_authselect_custom_profile; corrected duplicated STIG IDs in name (411080|411080|411090 -> 411080|411085|411090) and when condition (411085 or 411085 or 411090 -> 411080 or 411085 or 411090)
tasks/Cat3/*.yml: replaced CAT2 tag with CAT3 on all 15 Cat3 controls (selective `--tags CAT3` runs were silently empty)
tasks/Cat2/RHEL-09-653xxx.yml: 653025 task tag corrected from RHEL-09-653055 to RHEL-09-653025
tasks/Cat2/RHEL-09-654xxx.yml: 654245 shadow-audit task now uses rhel_09_654245 toggle and RHEL-09-654245 tag (previously both pointed at 654240)
Task name titles aligned verbatim to V2R7 XCCDF for 12 controls where the role title was inverted, cross-pasted from an adjacent rule, contained discussion text, or otherwise diverged: 213090 (storage->disable storing), 231170 (noexec->nosuid), 252015 (chrony package->chronyd service), 271080 (idle-delay->lock-delay), 431020 (SELinux targeted policy->faillock tally directory context), 611180 (pcsc-lite package->pcscd service), 651020/651025 (file integrity tool->cryptographic mechanisms audit tools), 214025 (locally installed packages->all software repositories), 251035 (discussion text->PPSM CAL rule title), 291010 (discussion text->disable USB mass storage), 411015 (added scope qualifiers removed)
RHEL-09-214025 find repo files task: removed use_regex:true (incompatible with glob pattern *.repo); the find module was silently returning zero files due to "nothing to repeat at position 0" regex error, leaving the gpgcheck=1 replace loop with an empty list. CAT-1 control now executes correctly.
RHEL-09-214025 Set gpgcheck task: path: "{{ item }}" -> path: "{{ item.path }}" (find returns stat dicts, not path strings); regexp tightened from ^gpgcheck (which matched only the word "gpgcheck" and left "gpgcheck=0" partially replaced as "gpgcheck=1=0") to ^gpgcheck\s*=.*$ to replace the full key=value line per the XCCDF fix-text. Surfaced by molecule failure once the find no-op was fixed.
RHEL-09-611195/611200 copy service file: added force:false to the copy task that clobbered the lineinfile-edited drop-in on every converge. Goss audit was flipping these two controls from pass to fail between converges because the copy ran before the lineinfile re-edit. Audit state now stable run-to-run.
RHEL-09-271065 ini_file: added no_extra_spaces:true so the dconf drop-in writes idle-delay=uint32 600 (no spaces around =) per dconf format convention, matching the audit goss content regex.

## Based on STIG V2R7 - 05 Jan 2026

# May 26 update for public release

Public issue #154 addressed thanks to @PrymalInstynct
Public issue # 157 and #158 addressed thanks to @hectoralicea
names updated to show PATCH/AUDIT where needed.
octal mode change dto symbolic
typo updates
tidy up var naming convention - prelim, discovered and unique naming
remove unused variables
connecting user test updated
vars moved to task rather than on blocks

# Initial

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

## 2.5.0 Based on STIG V2R5 07 August 2025

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
