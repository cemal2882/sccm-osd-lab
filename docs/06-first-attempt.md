# Chapter 6: First Attempt: PXE-Test

[← Chapter 5](05-osd-configuration.md) · [README](../README.md) · [Next: Second attempt →](07-second-attempt-success.md)

**Where we are:** DHCP, PXE, boot image, OS image, task sequence and deployment are ready.  
**In this chapter:** The first real test on a VM called `PXE-Test`. Two runs failed, but each got further: the first stopped at a full disk, the second hung at the end of Windows setup.

---

## Test VM

`PXE-Test`: BIOS, 2 GB RAM, Intel PRO/1000 MT on `intnet`, network first in boot order.

![Boot order](../images/06-attempt1/01-vm-boot.png)

![Network adapter](../images/06-attempt1/02-vm-network.png)

## 🔴 PXE-E52

The first boot stopped before getting a DHCP address:

![PXE-E52](../images/06-attempt1/03-pxe-e52.png)

DHCP service, binding and scope were all fine. A reboot fixed it. It was a temporary delay.

## ✅ PXE works

IP 192.168.10.100, CM01 answered and sent `pxeboot.com`, F12 started the network boot:

![PXE: IP received, F12](../images/06-attempt1/04-pxe-f12.png)

![Windows Boot Manager, server 192.168.10.30](../images/06-attempt1/05-boot-manager.png)

![Boot image loading over TFTP](../images/06-attempt1/06-wim-loading.png)

![Task Sequence Wizard asks for the PXE password](../images/06-attempt1/07-ts-password.png)

## 🔴 Run 1: 0x80070070

The task sequence failed at *Apply Operating System* with `0x80070070` (not enough disk space). F8 → `diskpart`:

![Disk 2397 MB](../images/06-attempt1/08-disk-full.png)

The disk was only **2397 MB**; the 4.4 GB image did not fit.
**Fix:** Resized the VM disk to 60 GB.

## 🔴 Run 2: stuck at "Getting ready"

All task sequence steps finished:

![Task sequence steps](../images/06-attempt1/09-ts-steps.png)

Then Windows stayed on "Getting ready" for a long time. VirtualBox showed **VMM load 92%**, guest load 7%: almost all CPU time went to the virtualization layer, not the VM.

![VMM load 92%](../images/06-attempt1/10-vmm-92.png)

![Getting ready](../images/06-attempt1/11-getting-ready.png)

I forced a reset. That was a mistake: the install was in the middle of Windows setup and could not recover.

![The computer restarted unexpectedly](../images/06-attempt1/12-restarted-unexpectedly.png)

## Decision: start over

The broken install left an **Unknown** device record in SCCM. With a record, the machine is no longer "unknown" and no longer sees the deployment. I deleted the record and planned a fresh VM.

![Unknown device record](../images/06-attempt1/13-unknown-device.png)

> **Lessons:**
> - Look up the error code (0x80070070 = disk full), then verify it (diskpart).
> - A forced reset in the middle of setup turns a hang into a broken install.
> - VMM 92% was a clue that the problem was in the virtualization layer. I solved it the next day.

**Next:** A fresh VM, and the real cause of the slowness.

---

**Microsoft Learn:** [Use PXE](https://learn.microsoft.com/en-us/intune/configmgr/osd/deploy-use/use-pxe-to-deploy-windows-over-the-network) · [Unknown computers](https://learn.microsoft.com/en-us/intune/configmgr/osd/get-started/prepare-for-unknown-computer-deployments) · [Monitor OS deployments](https://learn.microsoft.com/en-us/intune/configmgr/osd/deploy-use/monitor-operating-system-deployments)
