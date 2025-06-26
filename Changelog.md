# RHEL9STIG

## 2.3.0 Based on STIG V2R3 Jan28 2025

- RuleID Updates
- CCI Updates
- package removals/additions now dont skip if package present or not but run through giving correct state
- Upgraded several control to disruption_high
- authselect updates for faillock related controls
- var dictionaries renamed to allow easier overridding of variables
- RHEL-09-672010 - Becomes RHEL-09-215100
- RHEL-09-672020 - moved to CAT1 and approach changed to remediate inline with documentation
- RHEL-09-672030 - removed
- RHEL-09-171011 - Added
- RHEL-09-232103 - Added
- RHEL-09-232104 - Added
- RHEL-09-251025 - Added firewalld reload as issues seen for new connections
- RHEL-09-255064 - Added
- RHEL-09-255070 - Added
- RHEL-09-433016 - Added fapolicyd ading rules and testing - new var rhel9stig_allow_fapolicy_updates
- RHEL-09-610205 - title update
- RHEL-09-652035 - removed
- RHEL-09-653110 - added audit.rules
- RHEL-09-653130 - moved to 653xxx as 652035 no longer present
- RHEL-09-672035 - removed
- RHEL-09-672040 - removed
- RHEL-09-672045 - moved to 215105
- Addresses fix for RHEL-09-232200 on issue #115
- pre-commit updates include names

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
  - update audit sumamry output

## 1.2.0 Based on STIG V1R2 Jan24 2024

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

## 1.1.0 Based on STIG V1R1

Initial release
