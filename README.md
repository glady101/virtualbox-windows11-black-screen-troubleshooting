# Troubleshooting a Black Screen During Windows 11 Installation in VirtualBox

## Overview

This case study documents a black-screen issue encountered while installing **Windows 11 Enterprise Evaluation** in **Oracle VirtualBox 7.2.22** on an HP Windows 11 laptop. The VM started but did not visibly display the expected firmware or Windows installer screen. After reviewing the host configuration, virtual storage, and VirtualBox logs, I restored visible output by changing the VM graphics controller from **VBoxSVGA** to **VMSVGA**. I then completed Windows installation and confirmed that the VM could boot from its virtual disk and sign in successfully.

**Result:** Resolved and verified by installation, sign-in, and a subsequent successful boot.

> **Scope of conclusion:** The graphics-controller change was the decisive observed improvement, but the case does not prove a general VirtualBox defect or explain the lower-level cause of the graphics incompatibility. An earlier boot-order change remained in place during the successful test; therefore, the contribution of boot order to reaching the installer was not independently isolated.

## Lab Environment

| Component | Configuration |
| --- | --- |
| Host hardware | HP Laptop 15s-du3xxx |
| Host OS | Windows 11 Home Single Language (build 22631) |
| Host CPU | Intel Core i3-1115G4 (2 cores, 4 logical processors) |
| Host memory | 8 GB RAM |
| BIOS | UEFI; Intel Virtualization Technology enabled |
| Hypervisor environment | Windows hypervisor detected; VBS/HVCI running |
| Virtualization software | Oracle VirtualBox 7.2.22 r175245 |
| VM name | `jul` |
| Intended guest | Windows 11 Enterprise Evaluation (64-bit) |
| VM CPU / memory | 2 vCPUs / 4096 MB RAM |
| Firmware | EFI, Secure Boot enabled |
| Disk | 80 GB dynamically allocated VHD on SATA port 0 |
| Installation media | Windows 11 Enterprise Evaluation ISO on SATA port 1 |
| Original display configuration | VBoxSVGA, 128 MB VRAM, 3D acceleration disabled |
| Working display configuration | VMSVGA, 128 MB VRAM, 3D acceleration disabled |

> The host had limited free RAM during parts of the investigation. This was noted as an environmental constraint but **not established as the cause** of the black screen.

## Problem Statement

The new VM displayed a black screen on startup instead of showing the Windows installer. This was the first installation attempt for the VM named `jul`; its virtual hard disk did not contain an earlier working Windows installation. The ISO had been tried in previous installation attempts outside this VM.

## Troubleshooting Process

### 1. Verify host virtualization support

Intel virtualization was enabled in the system BIOS. VirtualBox's host information reported that hardware virtualization and nested paging were supported:

```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" list hostinfo
```

Windows also reported that a hypervisor was present:

```powershell
Get-CimInstance Win32_ComputerSystem |
    Select-Object HypervisorPresent
```

**Interpretation:** The CPU supported virtualization, but the Windows hypervisor was active. The existence of a Windows hypervisor was a diagnostic clue, not proof of the display failure.

### 2. Inspect VM configuration

```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" showvminfo "jul" --details
```

Important settings included EFI firmware, 2 virtual CPUs, 4096 MB RAM, VBoxSVGA graphics with 128 MB VRAM and 3D disabled, and an attached Windows installation ISO. Initially, the hard disk preceded the DVD in boot priority.

### 3. Inspect the virtual disk

```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" showmediuminfo "C:\Users\<HOST_USER>\VirtualBox VMs\jul\jul.vhd"
```

Representative output:

```text
Storage format: VHD
Format variant: dynamic default
Capacity:       81920 MBytes
Size on disk:   0 MBytes
State:          created
```

The VHD was a new, dynamically allocated 80 GB virtual disk. Because this was a fresh installation, the near-zero disk allocation was not treated as evidence of corruption.

### 4. Inspect the VirtualBox startup log

```powershell
$log = "C:\Users\<HOST_USER>\VirtualBox VMs\jul\Logs\VBox.log"

Get-Content $log -Tail 100

Select-String -Path $log -Pattern "VERR_|ERROR|WARNING|Guru|Boot|boot|EFI:|AHCI:|VD:|VHD:|Display:|VGA:|NEM:.*(error|failed)" |
    Select-Object -Last 80 |
    ForEach-Object { $_.Line }
```

Selected evidence:

```text
HM: HMR3Init: Attempting fall back to NEM: VT-x is not available
NEM: WHvCapabilityCodeHypervisorPresent is TRUE
AHCI: LUN#0: disk, PCHS=16383/16/63, total number of sectors 167772160
AHCI: LUN#1: CD/DVD
EFI: debug point SEC_PREMEM
EFI: debug point DXE_CORE
```

**Interpretation:** VirtualBox fell back to the Windows Hypervisor Platform execution path (NEM) rather than using VT-x directly. It recognized the virtual disk and DVD drive, and the EFI firmware began loading. These messages did not establish that the Windows installer had started or that NEM caused the black screen.

### 5. Test installation-media boot priority

With the VM powered off, I configured the DVD to boot before the hard disk:

```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" modifyvm "jul" --boot1 dvd --boot2 disk
```

**Result:** The screen remained black. Changing boot priority alone did not restore visible output.

### 6. Test a different graphics controller

With the VM fully powered off, I changed only the graphics controller:

```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" modifyvm "jul" --graphicscontroller vmsvga
```

**Result:** The VirtualBox/UEFI startup screen appeared. Windows Setup then became visible, allowing the installation to proceed.

The working graphics configuration retained **128 MB video memory** and **3D acceleration disabled**.

### 7. Complete installation

I completed the Windows Setup flow using the new 80 GB virtual disk, allowed setup to finish, and signed in to the Windows guest successfully.

### 8. Restore boot priority and verify an independent boot

Initially, a boot-order modification failed with:

```text
VBOX_E_INVALID_OBJECT_STATE (0x80bb0007)
The machine 'jul' is already locked for a session (or being unlocked)
```

This meant the VM was still locked by an active or closing VirtualBox session. After shutting down the guest and ensuring the VM was powered off, I verified its state:

```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" showvminfo "jul" --machinereadable |
    Select-String '^VMState='
```

Expected powered-off state:

```text
VMState="poweroff"
```

I then restored the disk-first boot order:

```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" modifyvm "jul" --boot1 disk --boot2 dvd
```

**Verification:** The VM booted successfully and Windows worked normally after the change.

## Before and After

| Test | Result |
| --- | --- |
| Initial VBoxSVGA configuration | Black screen |
| DVD-first boot, still using VBoxSVGA | Black screen |
| VMSVGA configuration, DVD-first boot | Firmware output and Windows Setup visible |
| Windows installation and sign-in | Successful |
| Restore disk-first boot and start again | Successful |

## Root-Cause Assessment

**Best-supported explanation:** A virtual graphics-controller compatibility problem affected visible startup output in this VM configuration. Switching from **VBoxSVGA** to **VMSVGA** was the decisive test that restored visible output.

**What the evidence does not prove:**

- That Windows Hypervisor Platform/NEM was the root cause. It was active, but no security feature needed to be disabled.
- That the ISO was damaged. The installer ultimately launched and completed installation.
- That the new VHD was faulty. The installation and subsequent boot succeeded.
- That graphics-controller behaviour would be identical across other hosts or VirtualBox versions.

## Lessons Learned

1. **Separate host virtualization capability from hypervisor ownership.** BIOS-enabled VT-x and VirtualBox's NEM fallback can coexist when Windows controls the hypervisor.
2. **Read startup events, not only shutdown statistics.** The last lines of `VBox.log` primarily described teardown and did not explain the initial blank display.
3. **Treat a new dynamic disk's small physical size in context.** An empty VHD is normal before a first installation.
4. **Change one variable at a time.** Changing graphics after a boot-order-only test made the observed difference clear.
5. **Avoid disabling host security controls without evidence.** The working solution required no VBS/HVCI changes.
6. **Make configuration changes only when the VM is powered off.** The lock error was a state-management issue, not an installation failure.
7. **Verify the entire outcome.** Seeing Windows Setup is progress; completing installation and booting from the disk is stronger evidence of a successful fix.

## Commands at a Glance

```powershell
$vbox = "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe"

& $vbox list hostinfo
& $vbox showvminfo "jul" --details
& $vbox showmediuminfo "C:\Users\<HOST_USER>\VirtualBox VMs\jul\jul.vhd"

# Run only while VM is fully powered off:
& $vbox modifyvm "jul" --boot1 dvd --boot2 disk
& $vbox modifyvm "jul" --graphicscontroller vmsvga

# Verify state before changing boot order again:
& $vbox showvminfo "jul" --machinereadable | Select-String '^VMState='
& $vbox modifyvm "jul" --boot1 disk --boot2 dvd
```

## Conclusion

The black-screen issue was resolved by changing the VM graphics controller from VBoxSVGA to VMSVGA. Windows Setup subsequently appeared, the guest installation completed, and the virtual machine successfully booted from its hard disk. The investigation also found active Windows hypervisor/NEM use, but did not require disabling virtualization-based security or making speculative host-level changes.

---

*This is a hands-on troubleshooting record. Machine-specific paths have been anonymized for public sharing.*
