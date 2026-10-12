# Home Lab: Active Directory, RBAC, Group Policy & Audit Logging

A single-domain Active Directory lab built to practice identity and access management from both the admin and the detection side: design role-based access, enforce a security baseline, then verify it with the Windows event log.

## Environment
| Item | Detail |
|---|---|
| Hypervisor | UTM (x86_64 emulation on an Apple M2 Mac) |
| Server | Windows Server 2022 Standard Evaluation (Desktop Experience), `DC01` |
| Domain | `lab.local` (NetBIOS `LAB`), DNS and Global Catalog on DC01 |
| Network | Static IP on DC01, DNS pointed at itself |

## What I built
1. **Domain controller:** installed AD DS, promoted DC01 to a new forest (`lab.local`).
2. **Structure:** OUs for `IT`, `HR`, `Finance`, `Contractors`; test users placed by department.
3. **RBAC:** security groups hold permissions, users hold group membership (never direct file permissions).
4. **File shares:** `C:\Shares\HR`, `Finance`, `IT`. Inheritance removed, broad `Users` access deleted, each folder granted to one group only.
5. **Group Policy:** domain password and lockout policy, plus a login banner GPO.
6. **Auditing:** Advanced Audit Policy for credential validation, account management, security group management, logon, and account lockout.

## Access matrix
| Group | HR folder | Finance folder | IT folder |
|---|---|---|---|
| `Grp-HR-read` | Read & execute | None | None |
| `Grp-Finance-Modify` | None | Modify | None |
| `Grp-IT-Admins` | None | None | Full control |

Verified with **Effective Access** on each folder. Example: `mjones` (in `Grp-HR-read`) gets read-only on HR (traverse, list, read) and no access to IT or Finance. Denied results are shown as well as allowed ones.

HR effective access: https://drive.google.com/file/d/1D3wCA8LPbSN8FCRSX6t8cHPcD-iurH3K/view?usp=sharing 

IT effective access: https://drive.google.com/file/d/1JWpkNEQfuZgePTbQyHEyhdymdzCKh1Yj/view?usp=sharing 

## Security baseline (Group Policy)
- Password: minimum length, complexity enabled, maximum age set
- Lockout: 5 invalid attempts, 15-minute lockout duration and reset window
- Login banner: "Authorized Use Only" notice
- Verified with `gpupdate /force` and `gpresult /r`

## Detection: what the logs showed
I deliberately generated bad logons, a lockout, an account creation, and a group change, then found each in **Event Viewer > Security**. Times are DC01's local clock.

| Time | Event ID | What it shows |
|---|---|---|
| 5:17:28 PM | 4720 | Account `mjones` created by `LAB\Administrator` |
| 6:41:49 PM | 4625 | Failed logon for `mjones`: unknown user name or bad password |
| 6:44:31 PM | 4740 | `mjones` locked out (caller computer `DC01`) |
| 6:48:16 PM | 4625 | Further attempt rejected: account locked out |
| 7:33:34 PM | 4728 | `tuser` added to `Grp-HR-read` |
| 7:37:07 PM | 4729 | `tuser` removed from `Grp-HR-read` by `LAB\Administrator` |

Takeaway: repeated bad passwords produced 4625 events, the lockout policy stopped the guessing after the threshold (4740), and later attempts were refused. Group membership changes are recorded with who made them, which is how access changes get reviewed.

4625: https://drive.google.com/file/d/1g-WeggSWSPwnonxabQKyskl4v62j5Epk/view?usp=sharing

4740: https://drive.google.com/file/d/1Di97HXD9oV8CT7Bkhd09ECDiIltquvYJ/view?usp=sharing

4728: https://drive.google.com/file/d/11xlXmRz3X6ElfYwZFoMxIwkrlRoavcA3/view?usp=sharing

4729: https://drive.google.com/file/d/1mps_i2S2_U5bxAcHCRvS5vhtdj7p9Bbj/view?usp=sharing

## Limitations
- One VM only: DC01 also hosts the shares. Real environments separate file servers from domain controllers.
- No domain-joined client: access was tested with Effective Access, not real logons from member machines.
- Failed logons were generated locally on DC01, so the source address is not a remote one.
- Auditing is local to DC01; there is no SIEM or log forwarding.

## What I learned
- Permissions belong on groups, and least privilege means removing broad default entries rather than only adding specific ones.
- A control isn't done until it is verified (`gpresult`, Effective Access, event logs).
- Which Event IDs map to logons, lockouts, account changes, and group changes.

## Possible next steps
- Add a domain-joined Windows client and test real logons
- Forward events to a SIEM (Wazuh or Splunk) and write detection rules for 4625 bursts and 4740
- Add a second group per folder (for example `Grp-HR-Modify`) to show separate read and write roles
