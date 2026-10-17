# Device Lifecycle Management with Microsoft Intune (MDM)

**Project 2 of 5: Help Desk Portfolio Series**

Simulates the full lifecycle of a corporate Windows 11 laptop the way a help desk or endpoint team handles it: **enroll → secure → configure → deploy apps → hand to a new hire → offboard**. Built in a Microsoft Intune tenant with a Windows 11 Enterprise VM.

---

## Table of Contents

1. [Problem Statement](#problem-statement)
2. [Environment](#environment)
3. [Lifecycle Overview](#lifecycle-overview)
4. [Phase A: Tenant, Users, and Groups](#phase-a-tenant-users-and-groups)
5. [Phase B: Enroll the Windows 11 Device](#phase-b-enroll-the-windows-11-device)
6. [Phase C: Compliance Policy and BitLocker](#phase-c-compliance-policy-and-bitlocker)
7. [Phase D: Configuration Profile and Win32 App](#phase-d-configuration-profile-and-win32-app)
8. [Phase E: New-Hire Provisioning](#phase-e-new-hire-provisioning)
9. [Phase F: Offboarding (Wipe and Cleanup)](#phase-f-offboarding-wipe-and-cleanup)
10. [Errors and Troubleshooting](#errors-and-troubleshooting)
11. [Key Concepts](#key-concepts)
12. [Skills Demonstrated](#skills-demonstrated)
13. [Project Status](#project-status)

---

## Problem Statement

A growing company hires a new employee every week. Each person needs a laptop that is **encrypted, patched, locked down, and loaded with the right apps** before their first login, and when they leave, the device must be **wiped and removed from management** so no company data walks out the door.

Doing this by hand (imaging, installing apps, checking settings one PC at a time) does not scale. Help desk teams solve it with **Mobile Device Management (MDM)**. This project builds that workflow end to end in Microsoft Intune and documents every step, including the things that broke along the way.

**Scenario:** New hire `newhire01` joins the company. IT enrolls their Windows 11 laptop, enforces security policy, auto-installs required software, verifies compliance, and later offboards the device completely.

---

## Environment

| Component | Detail |
|---|---|
| MDM platform | Microsoft Intune (free trial tenant, 30 days) |
| Identity | Microsoft Entra ID (Premium P2 trial, required for automatic MDM enrollment) |
| Tenant | `jayceintunelab26.onmicrosoft.com` (org: jayceaguilarhomelab) |
| Admin account | Global admin created at tenant signup |
| Test user | `newhire01` (Intune license assigned) |
| Groups | `Lab-NewHires` (users), `Lab-NewHire-Devices` (devices), both Security / Assigned |
| Endpoint | Windows 11 Enterprise Evaluation VM (build 26100) |
| Hypervisor | Oracle VirtualBox 7.2.2 |
| VM hardware | 4 GB RAM, 64 GB disk, TPM 2.0, UEFI, Secure Boot |
| Policies | `Win11-Baseline-Compliance`, `Win11-BitLocker-Policy`, `Win11-Security-Baseline` |
| App | 7-Zip 26.04 (x64), packaged as a Win32 app (`.intunewin`) |
| VM snapshots | `Pre-Enroll-00BE`, `Enrolled-Clean`, `Step-E-Provisioned` (rollback points) |

![VM hardware settings](screenshots/b-vm-hardware-settings.png)

> **Why Enterprise and not Home?** Windows 11 Home cannot join Entra ID or be managed by Intune the way companies need. Businesses deploy Pro or Enterprise, so the lab uses the free Enterprise evaluation.

---

## Lifecycle Overview

| Phase | Stage | What happens |
|---|---|---|
| A | Tenant setup | Create tenant, user, license, security groups |
| B | Enrollment | Join VM to Entra ID and auto-enroll into Intune |
| C | Compliance | Define "healthy device" and enforce BitLocker |
| D | Configuration | Push security settings and deploy a Win32 app |
| E | Provisioning | Verify the new hire's device is compliant with apps installed |
| F | Offboarding | Wipe device, remove records, confirm 0 enrolled devices |

---

## Phase A: Tenant, Users, and Groups

Created a new Intune tenant from the free trial, then built the identities the lab depends on.

**1. Tenant and admin center.** The Microsoft 365 admin center is where users, licenses, and groups are created.

![M365 admin center home](screenshots/a-m365-admin-home.png)

**2. Create the new hire.** User `New Hire 01` with an Intune license assigned. Without a license, the user cannot enroll a device.

![Create new hire user](screenshots/a-create-newhire-user.png)

![Active users](screenshots/a-m365-active-users.png)

**3. Security groups.** `Lab-NewHires` holds users and `Lab-NewHire-Devices` holds devices. Policies are assigned to groups, never to individuals, so onboarding becomes "add to group."

![Security group in M365](screenshots/a-m365-security-group.png)

![Groups in Intune](screenshots/a-intune-groups.png)

---

## Phase B: Enroll the Windows 11 Device

**1. Build the VM.** Windows 11 Enterprise Evaluation with TPM 2.0, UEFI, and Secure Boot enabled, matching what Windows 11 and BitLocker require.

![Windows install](screenshots/b-vm-windows-install.png)

**2. Enable automatic enrollment.** Under MDM user scope, set **All** so any user who signs in with a work account is enrolled automatically. This needs Entra ID Premium.

![MDM user scope](screenshots/b-mdm-user-scope-all.png)

**3. Join during OOBE.** At "Let's set things up for your work or school," signed in as `newhire01`. This performs an Entra join and triggers Intune enrollment in one step. I took a snapshot (`Pre-Enroll-00BE`) at this screen so the enrollment could be repeated.

![OOBE sign-in](screenshots/b-vm-oobe-signin.png)

**4. Verify on the device.** Settings → Accounts → Access work or school shows the account connected to the Entra tenant and **Managed by** the organization.

![Access work or school](screenshots/b-vm-access-work-school.png)

**5. Verify in Intune.** The device appears in Devices → All devices, managed by Intune, ownership Corporate.

![Device enrolled](screenshots/b-device-enrolled-compliant.png)

> **Watch out:** the "Compliant" badge here is misleading. Intune's built-in default compliance policy marks any device with no real policy as compliant. The first device overview shows the truth: no policies applied, no apps, and a real policy reporting Not Compliant.

![First check-in](screenshots/b-first-checkin-noncompliant.png)

---

## Phase C: Compliance Policy and BitLocker

A **compliance policy** defines what a healthy device looks like. Anything that fails is marked noncompliant, which a company can use to block access to email and files.

**1. Create `Win11-Baseline-Compliance`** for Windows 10 and later.

| Requirement | Setting |
|---|---|
| BitLocker | Require |
| Minimum OS version | 10.0.22621.0 |
| Firewall | Require |
| Antivirus | Require |
| Antispyware | Require |
| Microsoft Defender Antimalware | Require |
| Real-time protection | Require |

![Compliance health settings](screenshots/c-compliance-settings-health.png)

![Compliance security settings](screenshots/c-compliance-settings-security.png)

![Policy list](screenshots/c-compliance-policy-list.png)

**2. Assign to the device group.** The policy targets `Lab-NewHire-Devices`, and the VM is a member of that group.

![Device group members](screenshots/c-device-group-members.png)

**3. Enforce BitLocker.** A compliance policy only *checks* encryption. It doesn't turn it on. A second policy, `Win11-BitLocker-Policy` (Endpoint security → Disk encryption), enables silent encryption using the TPM.

| BitLocker setting | Value |
|---|---|
| Require Device Encryption | Enabled |
| Encryption type | Used Space Only |
| TPM startup | Allow TPM |
| Startup key / PIN | Not allowed (silent, no user prompt) |
| Recovery | 48-digit recovery password, rotation on Entra-joined devices |
| Assignment | Lab-NewHire-Devices (Include) |

![BitLocker policy settings](screenshots/c-bitlocker-policy-review.png)

![BitLocker policy assignment](screenshots/c-bitlocker-policy-assignments.png)

**4. Verify.** After sync, all seven BitLocker settings report Succeeded, and `manage-bde -status` on the VM shows Protection **On** with TPM and recovery password protectors.

![BitLocker policy succeeded](screenshots/c-bitlocker-policy-succeeded.png)

![manage-bde protection on](screenshots/c-bitlocker-protection-on.png)

**5. Confirm compliance.** The device reports Compliant against the policy, and per-setting status shows each requirement individually.

![Compliance device status](screenshots/c-compliance-device-status.png)

![Sync policies succeeded](screenshots/c-sync-policies-succeeded.png)

---

## Phase D: Configuration Profile and Win32 App

A **configuration profile** pushes settings onto the device. Compliance *measures*, configuration *changes*.

**1. Create `Win11-Security-Baseline`** (Settings catalog), assigned to `Lab-NewHire-Devices`.

| Setting | Value |
|---|---|
| Removable Disks: Deny read access | Enabled |
| Removable Disks: Deny write access | Enabled |
| Minimum password length | 8 |
| Max inactivity before device lock | 15 |

Blocking removable storage stops users from copying company data to USB drives.

![Security baseline settings](screenshots/d-security-baseline-settings.png)

**2. Package 7-Zip as a Win32 app.** Intune cannot deploy a raw `.msi` as a Win32 app. It needs the `.intunewin` format, created with Microsoft's Win32 Content Prep Tool (`IntuneWinAppUtil.exe`).

![Intunewin package](screenshots/d-intunewin-package.png)

**3. Create the app in Intune.** Apps → Windows → Add → Windows app (Win32), then upload the `.intunewin` file and fill in app information.

![Win32 app info](screenshots/d-win32-app-info.png)

| Program setting | Value |
|---|---|
| Install command | `msiexec /i "7z2604-x64.msi" /qn` |
| Uninstall command | `msiexec /x "7z2604-x64.msi" /qn` |
| Requirements | 64-bit OS, Windows 10 1607 or later |
| Detection rule | File exists: `C:\Program Files\7-Zip` |
| Assignment | **Required**, Included group `Lab-NewHire-Devices`, available as soon as possible, installation deadline as soon as possible |

`/qn` runs the installer silently so the user never sees a prompt. The detection rule is how Intune knows whether the app is already installed, so it doesn't reinstall on every check-in.

![Win32 app review](screenshots/d-win32-app-review.png)

The finished app's Properties page confirms the requirements, detection rule, and the **Required** assignment to `Lab-NewHire-Devices` (status Active).

![Win32 app properties and assignment](screenshots/d-win32-app-properties-assignment.png)

**4. Verify all three configuration policies applied.** BitLocker (device and user context) and the Security Baseline all show Succeeded.

![Device configuration succeeded](screenshots/d-device-config-3-succeeded.png)

---

## Phase E: New-Hire Provisioning

The goal: the device is compliant and the app installs without anyone touching the laptop.

**1. Overview.** Configuration policies Succeeded 3, compliance Compliant 2, managed apps Installed 1.

![Device overview](screenshots/e-device-overview.png)

**2. Managed apps.** 7-Zip 26.04 shows *Required install* with status **Installed**.

![Managed apps](screenshots/e-managed-apps-7zip-installed.png)

**3. On the device.** 7-Zip is present in `Program Files\7-Zip`, installed by Intune with no user action.

![7-Zip on the VM](screenshots/e-7zip-installed-on-vm.png)

At this point I took the `Step-E-Provisioned` snapshot so the wipe in Phase F could be rehearsed or repeated.

---

## Phase F: Offboarding (Wipe and Cleanup)

When an employee leaves, the device must be reset and removed from management.

**1. Choose the right action.** *Retire* removes company data but is **not supported on Entra-joined Windows devices**, so the correct action is **Wipe**. The option used was **Reset device and remove all data**.

![Wipe dialog](screenshots/f-wipe-dialog.png)

**2. Watch the device reset.** The VM restarts into "Resetting this PC" and returns to out-of-box setup, with all company data, the work account, and the installed app gone.

![VM resetting](screenshots/f-vm-resetting-pc.png)

**3. Clean up stale records.** A wipe does not delete the device object. I removed it from Intune, then from Entra ID (otherwise a ghost record stays behind).

![Entra device delete](screenshots/f-entra-device-delete.png)

**4. Verify the offboarding.** Intune All devices shows 0 devices, the Entra devices overview shows 0, and Tenant status shows **Total enrolled devices: 0**.

![Intune all devices empty](screenshots/f-all-devices-empty.png)

![Entra devices overview](screenshots/f-entra-devices-empty.png)

![Tenant status 0 devices](screenshots/f-tenant-status-0-devices.png)

---

## Errors and Troubleshooting

Real problems hit during the build, with cause and fix.

### 1. Developer sandbox rejected / consumer account blocked
- **Symptom:** The Microsoft 365 Developer Program said I did not qualify, and personal accounts could not open the admin portals ("consumer users without business presence").
- **Cause:** Admin centers require a work or school identity, not a personal Microsoft account.
- **Fix:** Signed up for the standalone Intune free trial, which creates a new tenant and an admin identity.

### 2. Automatic enrollment unavailable
- **Symptom:** MDM user scope could not be set for automatic enrollment.
- **Cause:** Automatic MDM enrollment requires Entra ID Premium.
- **Fix:** Activated the Entra ID P2 trial, then set MDM user scope to **All**.

![MDM user scope](screenshots/b-mdm-user-scope-all.png)

### 3. Policies not reaching the device
- **Symptom:** After sync, Policies showed *Not started / No policy updates*, device configuration showed *No items found*, and the compliance policy showed 0 devices.
- **Cause:** The compliance policy was assigned only to the user group (`Lab-NewHires`, 0 devices), and the device was not yet in a device group targeted by any policy.
- **Fix:** Created a device group, added the VM, and assigned the policies to `Lab-NewHire-Devices`. Synced again.

![Sync not started](screenshots/ts-sync-policies-not-started.png)

![Config no items](screenshots/ts-config-no-items.png)

![Compliance 0 devices](screenshots/ts-compliance-policy-0-devices.png)

![Assigned to user group](screenshots/ts-compliance-assigned-to-user-group.png)

![Fixed assignment](screenshots/ts-compliance-assignment-device-group.png)

### 4. Stale token breaks sync
- **Symptom:** Settings showed *Sign in again to fix your work or school account*, and the sync status reported **Policies: Failed (0 of 1 succeeded)**.
- **Cause:** The Entra sign-in token on the device had expired, so the device could not authenticate to Intune.
- **Fix:** Signed out and back in as `newhire01`, then forced a sync.

![Stale token](screenshots/ts-stale-token-signin-again.png)

![Sync failed](screenshots/ts-sync-policies-failed.png)

### 5. BitLocker error 0x80310030
- **Symptom:** BitLocker stayed at *Protection Off*, and adding the TPM protector failed with `0x80310030`: BitLocker detected bootable media (CD or DVD) in the computer. The per-setting compliance report showed BitLocker **Error: 1**.
- **Cause:** The Windows install ISO was still attached to the VM's virtual optical drive.
- **Fix:** Removed the disk from the optical drive in VirtualBox (Storage → Empty), restarted, and synced. Protection turned **On**.

```powershell
manage-bde -status
Add-BitLockerKeyProtector -MountPoint "C:" -TpmProtector
Resume-BitLocker -MountPoint "C:"
```

![0x80310030](screenshots/ts-bitlocker-0x80310030.png)

![Per-setting BitLocker error](screenshots/ts-per-setting-bitlocker-error.png)

![ISO removed](screenshots/ts-virtualbox-iso-removed.png)

### 6. Security baseline profile never applied
- **Symptom:** `Win11-Security-Baseline` stayed missing from the device even after sync.
- **Cause:** The group was added under **Excluded** instead of **Included**. Exclusion always wins.
- **Fix:** Moved `Lab-NewHire-Devices` to **Included**. The profile then showed Succeeded.

### 7. 7-Zip did not install after a "successful" sync
- **Symptom:** Intune said sync was successful, but 7-Zip was missing.
- **Cause:** The device's last check-in was *before* the app was created. Apps are installed by the Intune Management Extension, which runs on its own check-in cycle.
- **Fix:** Forced a device-side sync (Settings → Accounts → Access work or school → Info → Sync). The extension installed 7-Zip within minutes.

```powershell
Get-Service IntuneManagementExtension
Restart-Service IntuneManagementExtension
```

### 8. Retire is not available for this device
- **Symptom:** The Retire action was unsupported for the enrolled VM.
- **Cause:** Retire is not supported on Entra-joined Windows devices.
- **Fix:** Used **Wipe** with *Reset device and remove all data*.

### 9. Misleading "Compliant" status
- **Symptom:** The device showed Compliant immediately after enrollment, before any policy existed.
- **Cause:** The built-in Default Device Compliance Policy marks devices with no real policy as compliant.
- **Fix:** Verified with the **per-setting status report** instead of the badge.

---

## Key Concepts

| Concept | Takeaway |
|---|---|
| **MDM** | Cloud-based management of devices: enroll, configure, secure, deploy apps, wipe. |
| **Entra join + auto-enroll** | Signing in with a work account during setup joins the device to Entra ID and enrolls it in Intune in one step. |
| **Compliance policy vs configuration profile** | Compliance *checks* state (healthy or not). Configuration *pushes* settings. A BitLocker requirement needs both. |
| **Group assignment** | Policies target groups. **Excluded beats Included**, and user groups and device groups behave differently. |
| **Intune Management Extension** | The agent that installs Win32 apps. It checks in on its own schedule, separate from policy sync. |
| **Win32 app packaging** | `.msi` or `.exe` becomes `.intunewin` via the Content Prep Tool, with install, uninstall, and detection rules. |
| **BitLocker and TPM** | TPM stores the encryption key, with a recovery password escrowed to Entra ID for the help desk. |
| **Retire vs Wipe vs Delete** | Retire removes company data only, Wipe factory-resets, Delete removes the record. Entra-joined Windows devices need Wipe. |
| **Stale objects** | Removing a device from Intune does not remove it from Entra ID. Clean up both. |
| **AD + GPO vs Entra + Intune** | On-prem uses Active Directory and Group Policy. Cloud uses Entra ID and Intune. Hybrid joins both (covered in Project 3). |

---

## Skills Demonstrated

- Microsoft Intune administration: enrollment, compliance, configuration profiles, app deployment, remote wipe
- Entra ID: automatic MDM enrollment, device and group management
- BitLocker deployment and TPM key protector troubleshooting (`manage-bde`, PowerShell)
- Win32 app packaging and silent deployment (`msiexec`)
- Group-based policy targeting and assignment troubleshooting
- Log and status analysis: per-setting reports, sync status, Intune Management Extension
- Device offboarding and tenant cleanup
- Technical documentation with evidence for every phase

---

## Project Status

**Complete.** All six lifecycle phases were built, tested, and documented, including nine real troubleshooting cases.

**Possible extensions:** Windows Autopilot for zero-touch provisioning, Conditional Access that blocks noncompliant devices, update rings, and Intune Company Portal self-service.

**Series:** Project 1 (Ticket Lifecycle and SLA) → **Project 2 (Device Lifecycle, this repo)** → Project 3 (Identity Lifecycle) → Project 4 (Wi-Fi/DNS/DHCP/VPN troubleshooting) → Project 5 (capstone).
