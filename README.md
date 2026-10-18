# Device Lifecycle Management with Microsoft Intune (MDM)

Microsoft Intune · Entra ID · Windows 11 Enterprise · BitLocker

Hi,

In this hands-on project I simulate the full lifecycle of a corporate Windows laptop: tenant setup, enrollment, compliance, configuration, app deployment, new-hire provisioning, and offboarding. Everything is built in Microsoft Intune against a real Windows 11 Enterprise VM, and every failure along the way is diagnosed and documented.

## Table of Contents

- [Project Overview](#project-overview)
- [How the Device Lifecycle Works](#how-the-device-lifecycle-works)
- [What Was Built](#what-was-built)
- [Phase A — Tenant, Users & Groups](#phase-a--tenant-users--groups)
- [Phase B — Enroll the Device](#phase-b--enroll-the-device)
- [Phase C — Compliance Policy & BitLocker](#phase-c--compliance-policy--bitlocker)
- [Phase D — Configuration Profile & Win32 App](#phase-d--configuration-profile--win32-app)
- [Phase E — New-Hire Provisioning](#phase-e--new-hire-provisioning)
- [Phase F — Offboarding](#phase-f--offboarding)
- [Lifecycle Report](#lifecycle-report)
- [Resolution Notes](#resolution-notes)
- [Key Concepts Demonstrated](#key-concepts-demonstrated)
- [Tools & Technologies](#tools--technologies)
- [Project Status](#project-status)

## Project Overview

The goal of this project was to simulate what it looks like when a help desk onboards and offboards company laptops with policy instead of by hand: every device encrypted, locked down, and loaded with the right apps before the employee's first login, and fully removed when they leave.

| Component | Detail |
|---|---|
| MDM Platform | Microsoft Intune (free trial tenant) |
| Identity | Microsoft Entra ID (Premium P2 trial, required for automatic enrollment) |
| Tenant | jayceintunelab26.onmicrosoft.com |
| Endpoint | Windows 11 Enterprise Evaluation VM (build 26100), VirtualBox 7.2.2 |
| VM Hardware | 4 GB RAM, 64 GB disk, TPM 2.0, UEFI, Secure Boot |
| Test User | newhire01 (Intune license) |
| Groups | Lab-NewHires (users), Lab-NewHire-Devices (devices) |
| Policies | Win11-Baseline-Compliance, Win11-BitLocker-Policy, Win11-Security-Baseline |
| App | 7-Zip 26.04 x64, packaged as a Win32 app |

![VM hardware settings](screenshots/b-vm-hardware-settings.png)

## How the Device Lifecycle Works

Tenant, user, and groups created ↓
Device signs in with a work account, Entra joins, and auto-enrolls into Intune ↓
Compliance policy defines "healthy" and BitLocker policy enforces encryption ↓
Configuration profile and required app are pushed to the device group ↓
Device checks in, reports compliant, and the app installs silently ↓
Employee leaves: device is wiped and the records are removed

Key terms:

| Term | Meaning |
|---|---|
| MDM | Cloud management of devices: enroll, configure, secure, deploy apps, wipe |
| Compliance Policy | Defines what a healthy device looks like and measures it |
| Configuration Profile | Pushes settings onto the device |
| Win32 App | An installer packaged as `.intunewin` so Intune can deploy it silently |
| Wipe | Factory-resets the device and removes company data |

## What Was Built

- A new Intune tenant with a licensed test user and user and device security groups
- Automatic MDM enrollment through Entra join during Windows setup
- A compliance policy requiring BitLocker, minimum OS, firewall, antivirus, antispyware, and Defender protections
- A BitLocker policy that silently encrypts the drive with the TPM and escrows the recovery key
- A security baseline that blocks USB storage and sets password and lock rules
- A Win32 app (7-Zip) packaged with the Content Prep Tool and deployed as Required
- A full offboarding: wipe, stale-record cleanup, and verification of zero enrolled devices
- 9 real issues diagnosed and resolved along the way

## Phase A — Tenant, Users & Groups

Before enrolling anything, the tenant needed a user to enroll and groups to target.

The Microsoft 365 admin center was used to create the tenant's users and groups.

![M365 admin center home](screenshots/a-m365-admin-home.png)

User `New Hire 01` was created with an Intune license assigned. Without a license the user cannot enroll a device.

![Create new hire user](screenshots/a-create-newhire-user.png)

![Active users](screenshots/a-m365-active-users.png)

Two security groups were created. Policies are assigned to groups, never individuals, so onboarding becomes "add to the group."

![Security group in M365](screenshots/a-m365-security-group.png)

![Groups in Intune](screenshots/a-intune-groups.png)

## Phase B — Enroll the Device

A Windows 11 Enterprise VM was built with TPM 2.0, UEFI, and Secure Boot, matching what BitLocker and Windows 11 require.

![Windows install](screenshots/b-vm-windows-install.png)

MDM user scope was set to **All**, so any user signing in with a work account is enrolled automatically.

![MDM user scope](screenshots/b-mdm-user-scope-all.png)

At the "Let's set things up for your work or school" screen, the VM was signed in as newhire01. This Entra-joins the device and enrolls it in Intune in one step. A snapshot (`Pre-Enroll-00BE`) was taken here so enrollment could be repeated.

![OOBE sign-in](screenshots/b-vm-oobe-signin.png)

Verified on the device under Settings → Accounts → Access work or school:

![Access work or school](screenshots/b-vm-access-work-school.png)

And in Intune under Devices → All devices:

![Device enrolled](screenshots/b-device-enrolled-compliant.png)

The "Compliant" badge at this stage is misleading. It comes from Intune's built-in default compliance policy. The device overview shows the real state: no policies applied, no apps, and a real policy reporting Not Compliant.

![First check-in](screenshots/b-first-checkin-noncompliant.png)

## Phase C — Compliance Policy & BitLocker

A compliance policy defines what a healthy device looks like. Anything that fails is marked noncompliant.

**Win11-Baseline-Compliance** settings:

| Requirement | Setting |
|---|---|
| BitLocker | Require |
| Minimum OS version | 10.0.22621.0 |
| Firewall | Require |
| Antivirus / Antispyware | Require |
| Microsoft Defender Antimalware | Require |
| Real-time protection | Require |

![Compliance health settings](screenshots/c-compliance-settings-health.png)

![Compliance security settings](screenshots/c-compliance-settings-security.png)

![Policy list](screenshots/c-compliance-policy-list.png)

The policy is assigned to the device group, and the VM was added as a member.

![Device group members](screenshots/c-device-group-members.png)

A compliance policy only checks encryption, it does not turn it on. A second policy, **Win11-BitLocker-Policy**, enforces it:

| Setting | Value |
|---|---|
| Require Device Encryption | Enabled |
| Encryption type | Used Space Only |
| TPM startup | Allow TPM, no startup key or PIN |
| Recovery | 48-digit recovery password, rotation on Entra-joined devices |
| Assignment | Lab-NewHire-Devices (Include) |

![BitLocker policy settings](screenshots/c-bitlocker-policy-review.png)

![BitLocker policy assignment](screenshots/c-bitlocker-policy-assignments.png)

After sync, all seven BitLocker settings reported Succeeded and `manage-bde -status` showed Protection On with TPM and recovery password protectors.

![BitLocker policy succeeded](screenshots/c-bitlocker-policy-succeeded.png)

![manage-bde protection on](screenshots/c-bitlocker-protection-on.png)

The device then reported Compliant, and the sync status showed all policies applied.

![Compliance device status](screenshots/c-compliance-device-status.png)

![Sync policies succeeded](screenshots/c-sync-policies-succeeded.png)

## Phase D — Configuration Profile & Win32 App

Compliance measures the device, a configuration profile changes it.

**Win11-Security-Baseline** (Settings catalog), assigned to Lab-NewHire-Devices:

| Setting | Value |
|---|---|
| Removable Disks: Deny read access | Enabled |
| Removable Disks: Deny write access | Enabled |
| Minimum password length | 8 |
| Max inactivity before device lock | 15 |

![Security baseline settings](screenshots/d-security-baseline-settings.png)

7-Zip was packaged as a Win32 app. Intune cannot deploy a raw `.msi` this way, so it was converted to `.intunewin` with Microsoft's Win32 Content Prep Tool (`IntuneWinAppUtil.exe`).

![Intunewin package](screenshots/d-intunewin-package.png)

![Win32 app info](screenshots/d-win32-app-info.png)

| Program Setting | Value |
|---|---|
| Install command | `msiexec /i "7z2604-x64.msi" /qn` |
| Uninstall command | `msiexec /x "7z2604-x64.msi" /qn` |
| Requirements | 64-bit OS, Windows 10 1607 or later |
| Detection rule | File exists: `C:\Program Files\7-Zip` |
| Assignment | Required, Included group Lab-NewHire-Devices |

`/qn` installs silently so the user never sees a prompt. The detection rule tells Intune whether the app is already installed.

![Win32 app review](screenshots/d-win32-app-review.png)

![Win32 app properties and assignment](screenshots/d-win32-app-properties-assignment.png)

All three configuration policies applied to the device:

![Device configuration succeeded](screenshots/d-device-config-3-succeeded.png)

## Phase E — New-Hire Provisioning

The goal was a device that is compliant with its required app installed and no one touching the laptop.

The device overview shows configuration policies Succeeded 3, compliance Compliant 2, and managed apps Installed 1.

![Device overview](screenshots/e-device-overview.png)

![Managed apps](screenshots/e-managed-apps-7zip-installed.png)

7-Zip is present in Program Files on the VM, installed by Intune with no user action.

![7-Zip on the VM](screenshots/e-7zip-installed-on-vm.png)

A snapshot (`Step-E-Provisioned`) was taken here so the wipe could be repeated.

## Phase F — Offboarding

When an employee leaves, the device must be reset and removed from management.

Retire is not supported on Entra-joined Windows devices, so **Wipe** was used with "Reset device and remove all data."

![Wipe dialog](screenshots/f-wipe-dialog.png)

The VM restarted into "Resetting this PC" and returned to out-of-box setup.

![VM resetting](screenshots/f-vm-resetting-pc.png)

A wipe does not delete the device record. It was deleted from Intune, then from Entra ID so no stale object remained.

![Entra device delete](screenshots/f-entra-device-delete.png)

Verification:

![Intune all devices empty](screenshots/f-all-devices-empty.png)

![Entra devices overview](screenshots/f-entra-devices-empty.png)

![Tenant status 0 devices](screenshots/f-tenant-status-0-devices.png)

## Lifecycle Report

| Phase | Goal | Outcome | Evidence |
|---|---|---|---|
| A | Tenant, user, groups | Complete | Active users, security groups |
| B | Enroll device | Complete | Access work or school, All devices |
| C | Compliance and BitLocker | Complete after 3 fixes | Per-setting status, `manage-bde` Protection On |
| D | Config profile and app | Complete after 2 fixes | Device configuration 3 Succeeded |
| E | Provisioning | Complete | Overview: Succeeded 3, Compliant 2, Installed 1 |
| F | Offboarding | Complete | Total enrolled devices: 0 |

Issues encountered: 9 of 9 resolved. Root causes were assignment targeting, an expired sign-in token, an attached ISO, and check-in timing, not misconfigured policy settings.

## Resolution Notes

**#1 Developer sandbox rejected, consumer account blocked**

Diagnostic findings: The Microsoft 365 Developer Program said the account did not qualify, and a personal Microsoft account could not open the admin portals ("consumer users without business presence").

Root cause: Admin centers require a work or school identity.

Resolution: Signed up for the standalone Intune free trial, which creates a new tenant and admin identity.

**#2 Automatic enrollment unavailable**

Diagnostic findings: MDM user scope could not be set for automatic enrollment.

Root cause: Automatic MDM enrollment requires Entra ID Premium.

Resolution: Activated the Entra ID P2 trial and set MDM user scope to All.

![MDM user scope](screenshots/b-mdm-user-scope-all.png)

**#3 Policies not reaching the device**

Diagnostic findings: Sync showed Policies "Not started / No policy updates," device configuration showed "No items found," and the compliance policy showed 0 devices.

![Sync not started](screenshots/ts-sync-policies-not-started.png)

![Config no items](screenshots/ts-config-no-items.png)

![Compliance 0 devices](screenshots/ts-compliance-policy-0-devices.png)

Root cause: The compliance policy was assigned only to the user group (Lab-NewHires, 0 devices), and the device was not in a device group targeted by any policy.

![Assigned to user group](screenshots/ts-compliance-assigned-to-user-group.png)

Resolution: Created a device group, added the VM, and assigned the policies to Lab-NewHire-Devices. Synced again.

![Fixed assignment](screenshots/ts-compliance-assignment-device-group.png)

Preventive note: Decide up front whether a policy targets users or devices, and keep the group type consistent with that decision.

**#4 Stale token breaks sync**

Diagnostic findings: Settings showed "Sign in again to fix your work or school account," and sync status reported Policies: Failed (0 of 1 succeeded).

![Stale token](screenshots/ts-stale-token-signin-again.png)

![Sync failed](screenshots/ts-sync-policies-failed.png)

Root cause: The Entra sign-in token on the device had expired, so the device could not authenticate to Intune.

Resolution: Signed out and back in as newhire01, then forced a sync.

**#5 BitLocker error 0x80310030**

Diagnostic findings: BitLocker stayed at Protection Off. Adding the TPM protector failed with `0x80310030`: BitLocker detected bootable media (CD or DVD) in the computer. The per-setting report showed BitLocker Error: 1.

```powershell
manage-bde -status
Add-BitLockerKeyProtector -MountPoint "C:" -TpmProtector
Resume-BitLocker -MountPoint "C:"
```

![0x80310030](screenshots/ts-bitlocker-0x80310030.png)

![Per-setting BitLocker error](screenshots/ts-per-setting-bitlocker-error.png)

Root cause: The Windows install ISO was still attached to the VM's virtual optical drive.

Resolution: Removed the disk from the optical drive in VirtualBox (Storage → Empty), restarted, and synced. Protection turned On.

![ISO removed](screenshots/ts-virtualbox-iso-removed.png)

Preventive note: Detach installation media once setup finishes.

**#6 Security baseline never applied**

Diagnostic findings: Win11-Security-Baseline was missing from the device after repeated syncs.

Root cause: The group was added under Excluded instead of Included. Exclusion always wins.

Resolution: Moved Lab-NewHire-Devices to Included. The profile then showed Succeeded.

**#7 7-Zip not installed after a "successful" sync**

Diagnostic findings: Intune reported sync successful, but 7-Zip was missing. The device's last check-in was earlier than the app's creation time.

Root cause: Win32 apps are installed by the Intune Management Extension, which checks in on its own cycle, separate from policy sync.

```powershell
Get-Service IntuneManagementExtension
Restart-Service IntuneManagementExtension
```

Resolution: Forced a device-side sync (Settings → Accounts → Access work or school → Info → Sync). The extension installed 7-Zip within minutes.

**#8 Retire not available**

Diagnostic findings: The Retire action was unsupported for the enrolled VM.

Root cause: Retire is not supported on Entra-joined Windows devices.

Resolution: Used Wipe with "Reset device and remove all data."

**#9 Misleading "Compliant" status**

Diagnostic findings: The device showed Compliant right after enrollment, before any policy existed.

Root cause: The built-in Default Device Compliance Policy marks devices with no real policy as compliant.

Resolution: Verified with the per-setting status report instead of the badge.

## Key Concepts Demonstrated

- Intune enrollment through Entra join and automatic MDM enrollment
- Compliance policy vs configuration profile, and why BitLocker needs both
- Group-based assignment: user vs device groups, Included vs Excluded (exclusion wins)
- BitLocker deployment with TPM and recovery key escrow, and `manage-bde` troubleshooting
- Win32 app packaging (`.intunewin`), silent install, and detection rules
- Intune Management Extension check-in behavior
- Retire vs Wipe vs Delete, and cleanup of stale Entra objects
- Reading sync status and per-setting reports to find root cause

## Tools & Technologies

Microsoft Intune · Microsoft Entra ID · Microsoft 365 Admin Center · Windows 11 Enterprise · Oracle VirtualBox · BitLocker · PowerShell · Win32 Content Prep Tool

## Project Status

✅ Complete. Tenant, enrollment, compliance, configuration, app deployment, provisioning, and offboarding finished and documented in Intune. 9 issues diagnosed and resolved.
