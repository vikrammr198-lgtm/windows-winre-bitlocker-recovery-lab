# Commands Used

Major commands used during the Windows WinRE & BitLocker troubleshooting lab.

> **Security:** Machine-specific identifiers such as volume GUIDs, BCD GUIDs, recovery keys, usernames, computer names, and serial numbers have been replaced with placeholders.

---

## 1. WinRE Diagnosis

### Check WinRE status

```cmd
reagentc /info
```

Used to check WinRE status, location, BCD information, and version.

### Enable WinRE

```cmd
reagentc /enable
```

Initially failed because the recovery partition was detected as BitLocker-encrypted. After resolving the issue, the command completed successfully.

### Register WinRE image

```cmd
reagentc /setreimage /path R:\Recovery\WindowsRE /target C:\Windows
```

Registers the `Winre.wim` location with Windows.

---

## 2. Disk & Partition Investigation

### Start DiskPart

```cmd
diskpart
```

### Inspect disks

```cmd
list disk
```

### Select system disk

```cmd
select disk 0
```

### Inspect partitions

```cmd
list partition
```

### Inspect volumes

```cmd
list volume
```

### Inspect a partition

```cmd
select partition <PARTITION_NUMBER>
detail partition
```

---

## 3. Create Dedicated WinRE Partition

### Check available shrink space

```cmd
select partition <WINDOWS_PARTITION>
shrink querymax
```

### Shrink Windows partition

```cmd
shrink desired=2048
```

Creates approximately 2 GB of available space.

### Create recovery partition

```cmd
create partition primary size=2048
```

### Format recovery partition

```cmd
format quick fs=ntfs label="Windows RE tools"
```

---

## 4. Configure GPT Recovery Partition

### Set Windows Recovery partition type

```cmd
select partition <WINRE_PARTITION>
set id=<WINDOWS_RECOVERY_PARTITION_GUID> override
```

Windows Recovery Partition GUID:

```text
DE94BBA4-06D1-4D40-A16A-BFD50179D6AC
```

### Set GPT attributes

```cmd
gpt attributes=<WINRE_GPT_ATTRIBUTES>
```

### Verify configuration

```cmd
detail partition
```

Expected:

```text
Type     : Windows Recovery Partition
Hidden   : Yes
Required : Yes
```

---

## 5. WinRE File Setup

### Temporarily assign drive letter

```cmd
assign letter=R
```

### Create WinRE directory

```cmd
mkdir R:\Recovery\WindowsRE
```

### Copy WinRE image

```cmd
robocopy C:\Windows\System32\Recovery R:\Recovery\WindowsRE Winre.wim /copyall
```

If the file required forced comparison/copy:

```cmd
robocopy C:\Windows\System32\Recovery R:\Recovery\WindowsRE Winre.wim /copyall /is /it
```

### Verify Winre.wim

```powershell
Get-ChildItem -Force "R:\Recovery\WindowsRE"
```

---

## 6. BitLocker Investigation

### Check BitLocker status

```cmd
manage-bde -status
```

### PowerShell BitLocker information

```powershell
Get-BitLockerVolume
```

### Display useful BitLocker properties

```powershell
Get-BitLockerVolume |
Select-Object MountPoint,VolumeType,VolumeStatus,ProtectionStatus,EncryptionPercentage |
Format-Table -AutoSize
```

### Inspect the WinRE volume

```powershell
Get-BitLockerVolume -MountPoint "<WINRE_VOLUME_PATH>" |
Format-List *
```

The investigation revealed that the dedicated WinRE partition had unexpectedly become BitLocker-encrypted.

---

## 7. Decrypt Only the WinRE Partition

```powershell
Disable-BitLocker -MountPoint "<WINRE_VOLUME_PATH>"
```

This was used **only on the dedicated WinRE partition**.

> C: and D: were intentionally left BitLocker-protected.

---

## 8. WinRE Log Analysis

### Search ReAgent log

```cmd
findstr /i "BitLocker partition Winre Recovery error failed" <REAGENT_LOG_PATH>
```

### View recent log entries

```powershell
Get-Content "<REAGENT_LOG_PATH>" |
Select-Object -Last 60
```

Log location:

```text
C:\Windows\Logs\ReAgent\ReAgent.log
```

Important findings included:

```text
Partition has bitlocker
```

```text
skip partition because it does not meet WinRE requirements
```

```text
No suitable partition could be found for installing WinRE
```

---

## 9. Filesystem Verification

### Check volume information

```cmd
fsutil fsinfo volumeinfo R:
```

### Verify WinRE files

```powershell
Get-ChildItem -Force "R:\Recovery\WindowsRE"
```

---

## 10. Final Cleanup

### Remove temporary drive letter

```cmd
diskpart
select disk 0
select partition <WINRE_PARTITION>
remove letter=R
exit
```

The partition itself was **not deleted**; only the temporary drive letter was removed.

---

## 11. Final Verification

### Verify WinRE

```cmd
reagentc /info
```

Expected:

```text
Windows RE status: Enabled
```

### Verify BitLocker

```cmd
manage-bde -status
```

Final objective:

```text
C:      → BitLocker Protected
D:      → BitLocker Protected
WinRE   → Separate and Unencrypted
```

---

## Command Summary

| Area | Main Tools / Commands |
|---|---|
| WinRE | `reagentc` |
| Disk Management | `diskpart` |
| Partition Management | `list`, `select`, `shrink`, `create`, `format` |
| GPT | `set id`, `gpt attributes` |
| BitLocker | `manage-bde`, `Get-BitLockerVolume`, `Disable-BitLocker` |
| File Operations | `mkdir`, `robocopy` |
| Filesystem | `fsutil` |
| Log Analysis | `findstr`, `Get-Content` |
| PowerShell | `Get-ChildItem` |

---

## Final Result

```text
WinRE                    → Enabled
Dedicated Recovery      → 2 GB
Recovery Partition      → Correct GPT type
Winre.wim               → Present
Recovery Partition      → Hidden
Recovery Partition      → Unencrypted
Windows C:              → BitLocker Protected
Data D:                 → BitLocker Protected
```
