# Chapter 3: SCCM Installation

[← Chapter 2](02-prerequisites-and-sql.md) · [README](../README.md) · [Next: DC license →](04-dc-license.md)

**Where we are:** AD is prepared, IIS, .NET, ADK and SQL Server are installed.  
**In this chapter:** Install Configuration Manager 2509 as a stand-alone primary site (`TL1`) with SQL, MP and DP on one server. Three problems on the way: offline prerequisites, failed SQL checks, and the "SQL RMO" error.

---

## Setup wizard

**Typical installation, stand-alone primary site:**

![Typical install](../images/03-sccm/01-typical.png)

### 🔴 Prerequisite download failed

![Prerequisite download error](../images/03-sccm/02-prereq-download-error.png)

**Cause:** The NAT adapter was off, so CM01 had no internet.
**Fix:** I had already downloaded the files with `setupdl.exe` to `C:\CMPrereqs`. I chose *Use previously downloaded files*.

![Previously downloaded files](../images/03-sccm/03-predownloaded.png)

Site code `TL1`, site name `Testlab Primary`:

![Site code](../images/03-sccm/04-site-code.png)

## 🔴 Prerequisite check: Failed

First run: *SQL Server service running account* and *Dedicated SQL Server instance* failed.

![First check: Failed](../images/03-sccm/05-prereq-failed.png)

- **Service account:** SQL ran as the default virtual account. I changed it to **Local System** (lab only; production should use a domain account or gMSA). This check passed.

![SQL service: Local System](../images/03-sccm/06-sql-local-system.png)

- **Index create memory:** The check wanted the default value (0).

![Index create memory warning](../images/03-sccm/07-index-memory-warning.png)

![Fixed with sp_configure, sysadmin confirmed](../images/03-sccm/08-index-memory-fix.png)

- ODBC Driver 18 version confirmed (18.5.1.1):

![ODBC 18.5.1.1](../images/03-sccm/09-odbc.png)

### Still failing

Now *Dedicated SQL Server instance*, *Required SQL Server Collation* and *SQL Server sysadmin rights* all failed.

![Still Failed](../images/03-sccm/10-prereq-still-failed.png)

But I knew all three were correct: collation was verified in chapter 2, sysadmin was just confirmed. **Three correct settings failing together means the problem is the connection, not the settings.**

### 🔍 Root cause: TCP/IP disabled

```powershell
Test-NetConnection cm01.testlab.local -Port 1433
```

![Port 1433 closed](../images/03-sccm/11-1433-closed.png)

`TcpTestSucceeded : False`. In SQL Server Developer edition, **TCP/IP is disabled by default**. Setup connects to SQL by FQDN over TCP, so it could not verify anything.

**Fix:** SQL Server Configuration Manager → Protocols for MSSQLSERVER → **TCP/IP: Enabled**, then restart the SQL service.

![TCP/IP enabled](../images/03-sccm/12-sql-tcpip.png)

### 🔴 SQL service stopped

After CM01 froze and restarted, SQL stayed **Stopped**:

![SQL service stopped](../images/03-sccm/13-sql-stopped.png)

**Fix:** Started the service and set it to **Automatic (Delayed Start)**.

### ✅ Warnings only

No more failures. The remaining warnings do not block setup:

- *WSUS on site server:* software updates are out of scope.
- *SQL Server Native Client:* SCCM 2509 uses ODBC 18.
- *SQL Server security mode:* Windows auth only, confirmed with `IsIntegratedSecurityOnly = 1`.
- *SQL Server process memory:* max memory kept at 3072 MB on purpose.

![Warnings only](../images/03-sccm/14-warnings-only.png)

## 🔴 "Setup could not install SQL RMO"

![SQL RMO error](../images/03-sccm/15-sql-rmo-error.png)

**Investigation:** `ConfigMgrSetup.log` showed the real failure was `msoledbsql.msi` (OLE DB Driver 19). Running it by hand showed the actual message:

![VC++ 14.34 required](../images/03-sccm/16-vcpp-required.png)

**Root cause:** OLE DB Driver 19 needs Visual C++ 2015–2022 **14.34 or newer**. The server had an older version.

**Fix:** Enabled NAT briefly, installed VC++ x64 and x86, disabled NAT, installed the OLE DB driver.

![VC++ and OLE DB install](../images/03-sccm/17-vcpp-install.png)

![VC++ 14.44 and OLE DB Driver 19 installed](../images/03-sccm/18-vcpp-verify.png)

## ✅ Setup complete

![Core setup has completed](../images/03-sccm/19-install-done.png)

> **Lessons:**
> - When several checks fail at once, look for what they have in common.
> - A test must follow the real path. My localhost test never used TCP.
> - The component in the error message (SQL RMO) may not be the real problem. Follow the log.

**Next:** Setup finished at night. The next morning, the site was not healthy.

---

**Microsoft Learn:** [Prerequisite checker](https://learn.microsoft.com/en-us/intune/configmgr/core/servers/deploy/install/prerequisite-checker) · [Setup Downloader](https://learn.microsoft.com/en-us/intune/configmgr/core/servers/deploy/install/setup-downloader) · [SQL Server support](https://learn.microsoft.com/en-us/intune/configmgr/core/plan-design/configs/support-for-sql-server-versions)
