# Device Lifecycle Management with Microsoft Intune (MDM)

Microsoft Intune · Entra ID · Windows 11 Enterprise · BitLocker

Hi,

In this hands-on project I simulate the full lifecycle of a corporate Windows laptop in Microsoft Intune: enrollment, compliance, configuration, app deployment, new-hire provisioning, and offboarding. Everything runs against a real Windows 11 Enterprise VM, and every failure along the way is diagnosed and documented.

## Table of Contents

- [Project Overview](#project-overview)
- [How the Device Lifecycle Works](#how-the-device-lifecycle-works)
- [Phase A — Tenant, Users & Groups](#phase-a--tenant-users--groups)
- [Phase B — Enroll the Device](#phase-b--enroll-the-device)
- [Phase C — Compliance & BitLocker](#phase-c--compliance--bitlocker)
- [Phase D — Configuration Profile & Win32 App](#phase-d--configuration-profile--win32-app)
- [Phase E — New-Hire Provisioning](#phase-e--new-hire-provisioning)
- [Phase F — Offboarding](#phase-f--offboarding)
- [Resolution Notes](#resolution-notes)
- [Key Concepts Demonstrated](#key-concepts-demonstrated)
- [Tools & Technologies](#tools--technologies)
- [Project Status](#project-status)

## Project Overview

The goal was to simulate how a help desk onboards and offboards company laptops with policy instead of by hand: every device encrypted, locked down, and loaded with the right apps before the employee's first login, and fully removed when they leave.

| Component | Detail |
|---|---|
| MDM Platform | Microsoft Intune (free trial tenant) |
| Identity | Microsoft Entra ID (Premium P2 trial, required for automatic enrollment) |
| Endpoint | Windows 11 Enterprise Evaluation VM (build 26100), VirtualBox 7.2.2, TPM 2.0, UEFI, Secure Boot |
| Test User | newhire01 (Intune license) |
| Groups | Lab-NewHires (users), Lab-NewHire-Devices (devices) |
| Policies | Win11-Baseline-Compliance, Win11-BitLocker-Policy, Win11-Security-Baseline |
| App | 7-Zip 26.04 x64 (Win32 app) |

## How the Device Lifecycle Works

Tenant, user, and groups created ↓
Device signs in with a work account, Entra joins, and auto-enrolls into Intune ↓
Compliance and BitLocker policies define and enforce a healthy device ↓
Security profile and required app pushed to the device group ↓
Device checks in, reports compliant, and the app installs silently ↓
Employee leaves: device is wiped and its records removed

## Phase A — Tenant, Users & Groups

A new Intune tenant was created, along with a licensed test user (`New Hire 01`) and two security groups. Policies target groups, never individuals, so onboarding becomes "add to the group."

![Active users](screenshots/a-m365-active-users.png)

![Groups in Intune](screenshots/a-intune-groups.png)

## Phase B — Enroll the Device

MDM user scope was set to **All** so any work-account sign-in enrolls automatically.

![MDM user scope](screenshots/b-mdm-user-scope-all.png)

At the "Let's set things up for your work or school" screen, the VM signed in as newhire01. This Entra-joins the device and enrolls it in Intune in one step. Confirmed under Settings → Accounts → Access work or school:

![Access work or school](screenshots/b-vm-access-work-school.png)

The "Compliant" badge at this point is misleading. It comes from Intune's default compliance policy. The device overview shows the real state: no policies, no apps, and a real policy reporting Not Compliant.

![First check-in](screenshots/b-first-checkin-noncompliant.png)

## Phase C — Compliance & BitLocker

**Win11-Baseline-Compliance** requires BitLocker, minimum OS 10.0.22621.0, firewall, antivirus, antispyware, Defender Antimalware, and real-time protection.

![Compliance settings](screenshots/c-compliance-settings-health.png)

A compliance policy only checks encryption, it doesn't turn it on. **Win11-BitLocker-Policy** enforces it silently with the TPM (Used Space Only, no PIN, 48-digit recovery password escrowed to Entra ID). `manage-bde -status` confirmed Protection On:

![manage-bde protection on](screenshots/c-bitlocker-protection-on.png)

## Phase D — Configuration Profile & Win32 App

Compliance measures the device, configuration changes it. **Win11-Security-Baseline** denies read and write on removable disks (USB), sets minimum password length to 8, and locks the device after 15 minutes of inactivity.

![Security baseline](screenshots/d-security-baseline-settings.png)

7-Zip was converted from `.msi` to `.intunewin` with the Win32 Content Prep Tool and deployed as **Required** to Lab-NewHire-Devices.

| Setting | Value |
|---|---|
| Install command | `msiexec /i "7z2604-x64.msi" /qn` |
| Uninstall command | `msiexec /x "7z2604-x64.msi" /qn` |
| Detection rule | File exists: `C:\Program Files\7-Zip` |

![Win32 app assignment](screenshots/d-win32-app-properties-assignment.png)

## Phase E — New-Hire Provisioning

The goal: a compliant device with its required app installed and no one touching the laptop. The overview shows configuration policies Succeeded 3, compliance Compliant 2, and managed apps Installed 1.

![Device overview](screenshots/e-device-overview.png)

7-Zip is present in Program Files on the VM, installed by Intune with no user action.

![7-Zip on the VM](screenshots/e-7zip-installed-on-vm.png)

## Phase F — Offboarding

Retire is not supported on Entra-joined Windows devices, so **Wipe** was used with "Reset device and remove all data."

![Wipe dialog](screenshots/f-wipe-dialog.png)

![VM resetting](screenshots/f-vm-resetting-pc.png)

A wipe does not delete the device record. It was removed from Intune and then from Entra ID. Tenant status confirms zero enrolled devices:

![Tenant status 0 devices](screenshots/f-tenant-status-0-devices.png)

## Resolution Notes

**#1 Policies not reaching the device**

Diagnostic findings: Sync showed Policies "Not started / No policy updates" and the compliance policy showed 0 devices.

![Sync not started](screenshots/ts-sync-policies-not-started.png)

Root cause: The policy was assigned only to the user group, and the device wasn't in a device group targeted by any policy.

Resolution: Created a device group, added the VM, and assigned the policies to Lab-NewHire-Devices.

![Fixed assignment](screenshots/ts-compliance-assignment-device-group.png)

**#2 Stale token breaks sync**

Diagnostic findings: Settings showed "Sign in again to fix your work or school account."

![Stale token](screenshots/ts-stale-token-signin-again.png)

Root cause: The Entra sign-in token had expired, so the device couldn't authenticate to Intune.

Resolution: Signed out and back in as newhire01, then forced a sync.

**#3 BitLocker error 0x80310030**

Diagnostic findings: Protection stayed Off, and adding the TPM protector failed with `0x80310030` (bootable media detected).

```powershell
manage-bde -status
Add-BitLockerKeyProtector -MountPoint "C:" -TpmProtector
Resume-BitLocker -MountPoint "C:"
```

![0x80310030](screenshots/ts-bitlocker-0x80310030.png)

Root cause: The Windows install ISO was still attached to the VM's virtual optical drive.

Resolution: Removed the disk from the optical drive, restarted, and synced. Protection turned On.

**#4 Other issues**

| Issue | Root Cause | Resolution |
|---|---|---|
| Developer sandbox rejected, consumer account blocked | Admin centers need a work or school identity | Used the standalone Intune free trial |
| Auto-enrollment unavailable | Requires Entra ID Premium | Activated the P2 trial |
| Security baseline never applied | Group was under Excluded, and exclusion wins | Moved it to Included |
| 7-Zip missing after "successful" sync | The Intune Management Extension checks in on its own cycle | Forced a device-side sync |
| Retire unsupported | Not supported on Entra-joined devices | Used Wipe |

## Key Concepts Demonstrated

- Entra join and automatic MDM enrollment
- Compliance policy vs configuration profile, and why BitLocker needs both
- Group-based assignment: user vs device groups, Included vs Excluded (exclusion wins)
- BitLocker with TPM and recovery key escrow, `manage-bde` troubleshooting
- Win32 app packaging, silent install, and detection rules
- Retire vs Wipe vs Delete, and stale Entra object cleanup

## Tools & Technologies

Microsoft Intune · Microsoft Entra ID · Microsoft 365 Admin Center · Windows 11 Enterprise · Oracle VirtualBox · BitLocker · PowerShell

## Project Status

✅ Complete. Enrollment, compliance, configuration, app deployment, provisioning, and offboarding finished and documented in Intune. 9 issues diagnosed and resolved.
