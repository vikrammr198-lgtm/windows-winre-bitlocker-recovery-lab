# Windows WinRE & BitLocker Recovery Lab

> Hands-on Windows System Administration project documenting the diagnosis and repair of a Windows Recovery Environment (WinRE) configuration issue on a GPT/UEFI system with BitLocker enabled.

---

## 📌 Project Overview

This project documents a real-world Windows system administration and troubleshooting exercise.

The objective was to restore and properly configure **Windows Recovery Environment (WinRE)** while keeping the primary Windows and data partitions protected by **BitLocker**.

During the process, a dedicated recovery partition was created and configured. However, Windows automatically enabled BitLocker on the newly created recovery partition, preventing WinRE from being enabled.

The issue was diagnosed using:

- Windows Command Prompt
- PowerShell
- DiskPart
- BitLocker management tools
- REAgentC
- Windows recovery logs

The final configuration successfully enabled WinRE on a dedicated, unencrypted recovery partition while leaving the Windows and data partitions encrypted.

---

## 🎯 Objectives

- Diagnose why Windows RE was disabled.
- Investigate the existing disk and partition structure.
- Create a dedicated WinRE recovery partition.
- Configure the partition using the correct GPT recovery type.
- Copy and register `Winre.wim`.
- Diagnose BitLocker interference.
- Decrypt **only the recovery partition**.
- Enable Windows Recovery Environment.
- Verify the final configuration.

---

## 🖥️ System Environment

| Component | Configuration |
|---|---|
| Operating System | Windows |
| Firmware | UEFI |
| Partition Style | GPT |
| Windows Partition | C: |
| Data Partition | D: |
| Recovery Partition | Dedicated 2 GB partition |
| File System | NTFS |
| Encryption | BitLocker |
| Recovery Environment | Windows RE |

---

# 1. 🚨 Initial Problem

Windows Recovery Environment was initially disabled.

Running:

```cmd
reagentc /info
```

showed:

```text
Windows RE status: Disabled
```

Attempting to enable it:

```cmd
reagentc /enable
```

returned:

```text
REAGENTC.EXE: Windows RE cannot be enabled on a volume
with BitLocker Drive Encryption enabled.
```

### Initial investigation

The following commands were used:

```cmd
reagentc /info
manage-bde -status
diskpart
list disk
list partition
list volume
```

---

# 2. 🔍 Initial Disk Investigation

The system used a GPT disk with the following general structure:

```text
Partition 1    System       260 MB
Partition 2    Reserved      16 MB
Partition 3    Primary     ~372 GB   C:
Partition 4    Recovery      2 GB    WinRE
Partition 6    Primary      ~99 GB   D:
Partition 7    Recovery    260 MB
```

The Windows partition and data partition were protected by BitLocker.

### Important constraint

The Windows partition **C:** was already encrypted with BitLocker.

Instead of decrypting C:, the solution was to create a **separate recovery partition** for WinRE.

---

# 3. 🛠️ Creating the Dedicated WinRE Partition

First, the Windows partition was examined to determine how much space could safely be reclaimed.

```cmd
diskpart

select disk 0
select partition 3
shrink querymax
```

A 2 GB space allocation was then created:

```cmd
shrink desired=2048
```

The new recovery partition was created:

```cmd
create partition primary size=2048
```

The partition was formatted:

```cmd
format quick fs=ntfs label="Windows RE tools"
```

---

# 4. ⚙️ Configuring the Recovery Partition

The new partition was configured with the Microsoft Windows Recovery partition GUID:

```cmd
select disk 0
select partition 4

set id=de94bba4-06d1-4d40-a16a-bfd50179d6ac override
```

The GPT attributes were configured:

```cmd
gpt attributes=0x8000000000000001
```

These attributes ensure that the recovery partition is treated as a required system recovery partition and does not normally receive a drive letter.

---

# 5. 📦 Installing WinRE

A temporary drive letter was assigned to the recovery partition:

```cmd
assign letter=R
```

The required directory structure was created:

```cmd
mkdir R:\Recovery\WindowsRE
```

The Windows Recovery Image was copied:

```cmd
robocopy C:\Windows\System32\Recovery R:\Recovery\WindowsRE Winre.wim /copyall /is /it
```

The `Winre.wim` file was approximately:

```text
1,193,768,406 bytes
```

The recovery image was then registered with Windows:

```cmd
reagentc /setreimage /path R:\Recovery\WindowsRE /target C:\Windows
```

---

# 6. ❌ WinRE Enable Failed Again

Although the recovery partition was correctly configured, running:

```cmd
reagentc /enable
```

still failed with:

```text
Windows RE cannot be enabled on a volume with
BitLocker Drive Encryption enabled.
```

At this point, the problem was no longer the partition layout.

Further investigation was required.

---

# 7. 🔎 Root Cause Analysis

Windows Recovery logs were examined:

```text
C:\Windows\Logs\ReAgent\ReAgent.log
```

The relevant log entries showed:

```text
Partition has bitlocker
```

followed by:

```text
skip partition because it does not meet WinRE requirements
```

and eventually:

```text
No suitable partition could be found for installing WinRE
```

### Root Cause

The newly created 2 GB recovery partition had unexpectedly become **BitLocker-encrypted**.

This was the key reason ReAgentC refused to use the partition.

---

# 8. 🔐 BitLocker Investigation

The BitLocker state was examined using:

```cmd
manage-bde -status
```

PowerShell was also used:

```powershell
Get-BitLockerVolume
```

The recovery volume was found to be encrypted.

Further inspection showed:

```text
VolumeStatus          : FullyEncrypted
ProtectionStatus      : On
EncryptionPercentage  : 100
VolumeType            : Data
```

The recovery partition had effectively been treated as a BitLocker-protected data volume even though its GPT partition type was correctly configured for Windows Recovery.

---

# 9. 🔓 Decrypting Only the Recovery Partition

The Windows and data partitions were intentionally left encrypted.

The recovery volume was identified using its volume GUID.

The BitLocker PowerShell cmdlet was used:

```powershell
Disable-BitLocker -MountPoint "\\?\Volume{WINRE-VOLUME-GUID}\"
```

The recovery partition entered:

```text
DecryptionInProgress
```

After decryption completed, the recovery volume was no longer reported as a BitLocker volume.

### Important

Only the **2 GB WinRE partition** was decrypted.

The following partitions remained protected:

```text
C:  → BitLocker enabled
D:  → BitLocker enabled
```

---

# 10. ✅ Verifying WinRE Files

The recovery image was verified:

```powershell
Get-ChildItem -Force "R:\Recovery\WindowsRE"
```

Result:

```text
Winre.wim
```

File size:

```text
1,193,768,406 bytes
```

This confirmed that the Windows Recovery Image was present and intact.

---

# 11. 🚀 Enabling Windows Recovery Environment

After the recovery partition was no longer BitLocker-encrypted:

```cmd
reagentc /enable
```

returned:

```text
REAGENTC.EXE: Operation Successful.
```

🎉 **Windows Recovery Environment was successfully enabled.**

---

# 12. 🔎 Final Verification

The final configuration was verified using:

```cmd
reagentc /info
```

Result:

```text
Windows RE status: Enabled

Windows RE location:
\\?\GLOBALROOT\device\harddisk0\partition4\Recovery\WindowsRE

BCD identifier:
<Recovery-GUID>

Windows RE Version:
10.0.26100.9444
```

The recovery partition was also verified using DiskPart:

```cmd
diskpart

select disk 0
select partition 4
detail partition
```

Expected configuration:

```text
Type     : de94bba4-06d1-4d40-a16a-bfd50179d6ac
Hidden   : Yes
Required : Yes
Attrib   : 0x8000000000000001
```

---

# 13. 💾 Final Disk Configuration

The final configuration was:

```text
GPT / UEFI Disk
│
├── Partition 1
│   └── EFI System Partition – 260 MB
│
├── Partition 2
│   └── Microsoft Reserved – 16 MB
│
├── Partition 3
│   └── Windows – ~372 GB
│       └── BitLocker Protected
│
├── Partition 4
│   └── Windows Recovery – 2 GB
│       └── Winre.wim
│       └── Unencrypted
│
├── Partition 6
│   └── Data – ~99 GB
│       └── BitLocker Protected
│
└── Partition 7
    └── Existing Recovery Partition – 260 MB
```

---

# 14. 🧪 Commands Used

## Windows Recovery

```cmd
reagentc /info
reagentc /enable
reagentc /setreimage /path R:\Recovery\WindowsRE /target C:\Windows
```

## BitLocker

```cmd
manage-bde -status
```

```powershell
Get-BitLockerVolume
```

```powershell
Disable-BitLocker -MountPoint "\\?\Volume{WINRE-VOLUME-GUID}\"
```

## DiskPart

```cmd
diskpart

select disk 0
select partition 3
shrink querymax
shrink desired=2048

create partition primary size=2048

format quick fs=ntfs label="Windows RE tools"

select partition 4

set id=de94bba4-06d1-4d40-a16a-bfd50179d6ac override

gpt attributes=0x8000000000000001

assign letter=R
```

## File Operations

```cmd
mkdir R:\Recovery\WindowsRE
```

```cmd
robocopy C:\Windows\System32\Recovery R:\Recovery\WindowsRE Winre.wim /copyall /is /it
```

## File System Verification

```powershell
Get-ChildItem -Force "R:\Recovery\WindowsRE"
```

```cmd
fsutil fsinfo volumeinfo R:
```

## Log Analysis

```powershell
Get-Content 'C:\Windows\Logs\ReAgent\ReAgent.log' |
Select-Object -Last 60
```

---

# 15. 🧠 Key Lessons Learned

### 1. Don't assume the error message tells the whole story

The initial error mentioned BitLocker, but it wasn't immediately obvious **which partition** was causing the problem.

Log analysis revealed that the recovery partition itself was being detected as BitLocker-encrypted.

### 2. Logs are critical for Windows troubleshooting

The file:

```text
C:\Windows\Logs\ReAgent\ReAgent.log
```

provided the evidence needed to identify the actual problem.

### 3. Partition type and encryption state are separate concepts

The partition could have the correct Windows Recovery GUID:

```text
DE94BBA4-06D1-4D40-A16A-BFD50179D6AC
```

while still being BitLocker-encrypted.

Both configurations needed to be correct.

### 4. Avoid unnecessary decryption

The Windows partition did not need to be decrypted.

The solution preserved BitLocker protection on:

```text
C:
D:
```

while keeping the dedicated WinRE partition unencrypted.

### 5. Verify after every major system change

The configuration was repeatedly verified using:

- `DiskPart`
- `manage-bde`
- PowerShell
- `ReAgentC`
- Windows logs

---

# 16. 🛡️ Safety Considerations

Disk and partition operations can cause data loss if performed incorrectly.

Important precautions:

- Confirm the correct disk before using DiskPart.
- Confirm the partition number before modifying it.
- Never blindly use `clean`.
- Never delete a partition without verifying its purpose.
- Do not modify BitLocker-protected partitions unnecessarily.
- Keep BitLocker recovery keys available before performing disk operations.
- Verify the final configuration with `reagentc /info`.

> **This repository documents a personal lab/troubleshooting exercise. Commands that modify partitions or encryption should not be executed blindly on another system.**

---

# 17. 📸 Screenshots

Screenshots documenting the troubleshooting process are stored in the `screenshots/` directory.

Recommended evidence:

| Screenshot | Evidence |
|---|---|
| `01-initial-reagentc.png` | WinRE initially disabled |
| `02-initial-disk-layout.png` | Initial partition layout |
| `03-bitlocker-status.png` | BitLocker status |
| `04-partition-created.png` | New 2 GB recovery partition |
| `05-winre-wim.png` | WinRE image copied |
| `06-reagent-log.png` | BitLocker root cause |
| `07-bitlocker-recovery-partition.png` | Recovery partition encryption |
| `08-decryption.png` | Recovery partition decryption |
| `09-reagentc-success.png` | WinRE successfully enabled |
| `10-final-verification.png` | Final configuration |

---

# 18. 🎓 Skills Demonstrated

- **Windows System Administration**
- **Disk & Partition Management**
- **GPT / UEFI Partitioning**
- **BitLocker Administration**
- **Windows Recovery Environment (WinRE)**
- **PowerShell**
- **Windows Command Prompt**
- **DiskPart**
- **Log Analysis**
- **Root-Cause Analysis**
- **System Troubleshooting**
- **Storage Management**
- **Risk-Aware System Configuration**

---

# 19. 📚 Technologies & Tools

```text
Windows
PowerShell
Command Prompt
DiskPart
BitLocker
REAgentC
NTFS
GPT
UEFI
WinRE
Winre.wim
Windows Event/Recovery Logs
```

---

## 🏁 Final Outcome

The Windows Recovery Environment was successfully restored and configured on a dedicated **2 GB recovery partition**.

The final state achieved:

```text
WinRE                  → ENABLED ✅
Dedicated Recovery     → YES ✅
Recovery Partition     → 2 GB ✅
Winre.wim              → PRESENT ✅
Recovery GUID          → CORRECT ✅
Recovery Partition     → HIDDEN ✅
Recovery Partition     → UNENCRYPTED ✅
Windows C:             → BitLocker PROTECTED ✅
Data D:                → BitLocker PROTECTED ✅
```

**Result: Windows Recovery Environment successfully restored without decrypting the primary Windows or data partitions.**
