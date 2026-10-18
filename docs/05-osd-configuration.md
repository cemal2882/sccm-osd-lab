# Chapter 5: OSD Configuration

[← Chapter 4](04-dc-license.md) · [README](../README.md) · [Next: First attempt →](06-first-attempt.md)

**Where we are:** A healthy SCCM site, published in AD.  
**In this chapter:** Prepare everything a blank machine needs. A blank machine asks three questions, and this chapter answers them in order:

1. **Path:** Who am I, who do I talk to? → DHCP, boundaries, PXE
2. **Content:** What do I install? → boot image, OS image, domain join account
3. **Instructions:** In what order? → task sequence, deployment

---

## 1. Path

### DHCP (DC1)

Scope `WDS-Lab`: 192.168.10.100–150, option 6 (DNS 192.168.10.10), option 15 (`testlab.local`).

![DHCP scope options](../images/05-osd/01-dhcp.png)

**No options 60/66/67.** Microsoft does not support controlling ConfigMgr PXE with DHCP options. DHCP, PXE server and client are on the same subnet, so CM01 hears the broadcast directly. On different subnets, routers would need **IP helpers**.

### Boundary and boundary group

WinPE finds its distribution point through the boundary group, based on its IP. Boundary `192.168.10.0` → group `LAB-BG` → site `TL1` + site system `cm01.testlab.local`.

![Boundary](../images/05-osd/02-boundary.png)

![Boundary group](../images/05-osd/03-boundary-group.png)

![Site system](../images/05-osd/04-bg-site-system.png)

### PXE on the distribution point

- **Enable unknown computer support:** machines not in SCCM can boot.
- **PXE responder without WDS:** SCCM's own PXE service (`SccmPxe`), no WDS needed.
- **Require a password:** not everyone can start a deployment.

![DP PXE settings](../images/05-osd/05-dp-pxe.png)

Firewall: UDP 67/68 (DHCP/PXE), 69 (TFTP), 4011 (PXE).

```powershell
New-NetFirewallRule -DisplayName "SCCM PXE" -Direction Inbound -Protocol UDP -LocalPort 67,68,69,4011 -Action Allow
```

![Firewall rule and SccmPxe running](../images/05-osd/06-firewall.png)

## 2. Content

### Boot image

SCCM built an x64 boot image (`TL100002`) from the ADK during setup.

- **Deploy this boot image from the PXE-enabled DP**
- **Enable command support (testing only):** F8 opens a command prompt in WinPE. I used it in chapter 6 to check the disk.

![Boot image: PXE](../images/05-osd/07-boot-image-pxe.png)

![F8 command support](../images/05-osd/08-boot-image-f8.png)

![Boot image distributed](../images/05-osd/09-boot-image-distributed.png)

### 🔴 Domain join account created disabled

I created `svc-djoin` instead of using Administrator, because its password is stored in the task sequence (least privilege).

![Password policy error](../images/05-osd/10-svc-djoin-error.png)

First "password does not meet requirements", then "account already exists". `New-ADUser` had created the account but left it **disabled** because the password failed the policy (min 14 characters + complexity).

![Password policy](../images/05-osd/11-password-policy.png)

```powershell
$p = Read-Host -AsSecureString "svc-djoin password"
Set-ADAccountPassword svc-djoin -Reset -NewPassword $p
Enable-ADAccount svc-djoin
```

![svc-djoin enabled (password hidden)](../images/05-osd/12-svc-djoin.png)

No extra rights. By default any user can join 10 computers (`ms-DS-MachineAccountQuota`), enough for the lab.

### Finding `install.wim`

I attached an old WDS lab disk (`winserver2022dcdisk2_1.vdi`) to CM01, brought it online and found `E:\Images\install.wim` (4.4 GB, one index: Windows 10 Pro).

![Attach disk to CM01](../images/05-osd/13-attach-disk.png)

![install.wim found](../images/05-osd/14-wim-found.png)

![Get-WimInfo](../images/05-osd/15-wim-index.png)

SCCM reads images only from a UNC path. I copied it to `C:\Sources\OS`, shared it as `\\CM01\Sources` (read-only), added it to SCCM (`TL100006`) and distributed it.

![Sources share](../images/05-osd/16-sources.png)

![OS image added](../images/05-osd/17-os-image.png)

## 3. Instructions

### Task sequence `Lab-Win10 deploy`

| Step | Setting |
| --- | --- |
| Boot image | Boot image (x64) |
| Install Windows | `1-Windows-10-Pro.wim`, index 1, partition and format |
| Configure Network | Join `testlab.local` with `TESTLAB\svc-djoin` |
| ConfigMgr client | Configuration Manager Client Package |
| State migration, updates, apps | Off |

![Task sequence info](../images/05-osd/18-ts-info.png)

![Install Windows](../images/05-osd/19-ts-windows.png)

![Domain join](../images/05-osd/20-ts-domain.png)

![Task sequence created](../images/05-osd/21-ts-done.png)

### Deployment

Collection **All Unknown Computers**, purpose **Available**, *Only media and PXE*.

Why Available: the install starts only when someone presses F12. **Required + Unknown Computers** could wipe any new machine that boots from the network.

![Deployment: collection](../images/05-osd/22-deploy-collection.png)

![Deployment: Available, media and PXE only](../images/05-osd/23-deploy-settings.png)

**Next:** Everything is configured. Time to boot a blank VM from the network.

---

**Microsoft Learn:** [Use PXE](https://learn.microsoft.com/en-us/intune/configmgr/osd/deploy-use/use-pxe-to-deploy-windows-over-the-network) · [Prepare site system roles](https://learn.microsoft.com/en-us/intune/configmgr/osd/get-started/prepare-site-system-roles-for-operating-system-deployments) · [Unknown computers](https://learn.microsoft.com/en-us/intune/configmgr/osd/get-started/prepare-for-unknown-computer-deployments) · [Boundary groups](https://learn.microsoft.com/en-us/intune/configmgr/core/servers/deploy/configure/boundary-groups) · [Manage OS images](https://learn.microsoft.com/en-us/intune/configmgr/osd/get-started/manage-operating-system-images) · [PXE troubleshooting (DHCP options)](https://learn.microsoft.com/en-us/troubleshoot/mem/configmgr/os-deployment/troubleshoot-pxe-boot-issues)
