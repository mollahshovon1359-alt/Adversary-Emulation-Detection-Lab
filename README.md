# Adversary-Emulation-Detection-Lab


## Objective

This project builds a controlled home lab environment to **emulate real-world adversary behaviors** and **engineer custom detection rules**.

Using an Apple Silicon (M1) MacBook Pro as the host, a Windows virtual machine was deployed as the target workstation. The lab demonstrates the complete lifecycle of a cyber attack:

1. Executing malicious techniques using the **MITRE ATT&CK-aligned Atomic Red Team** framework
2. Routing live system telemetry via the **Splunk Universal Forwarder**
3. Analyzing and validating attack signatures within **Splunk Enterprise**

---

## Infrastructure Setup

### Component Matrix

| Component | Details |
|-----------|---------|
| **Host System** | Apple Silicon MacBook Pro (M1, macOS, 8 GB RAM) |
| **SIEM Platform** | Splunk Enterprise (localhost:8000) |
| **Hypervisor** | UTM (Open-source, Apple Silicon optimized, 4 GB, NAT) |
| **Guest Workstation** | Windows 11 Enterprise (via UTM) |
| **Telemetry Agents** | Microsoft Sysmon + Splunk Universal Forwarder (Windows) |

### Data Flow Architecture

```
WINDOWS GUEST VM (UTM)
  [Atomic Red Team] → Emulates Malicious Actions
         ↓
  [Microsoft Sysmon] → Generates Operational Logs
         ↓
  [Splunk Universal Forwarder]
         |
         | (Routed via UTM Shared Network/NAT Interface)
         ↓
macOS HOST SYSTEM
  [Splunk Enterprise SIEM] → Listens on Port 9997
         ↓
  [macOS Web Browser] → UI Analysis (Port 8000)
```
[![Alt Text]
---

## Lab Setup

### 3.1: Splunk Enterprise Server Installation on macOS (Host)

Because Splunk does not distribute a native ARM64 binary for macOS, installation relies on the **Rosetta 2** dynamic binary translation engine.

**Step 1 — Install Rosetta 2:**
```bash
softwareupdate --install-rosetta --agree-to-license
```

**Step 2 — Download and Mount:**
- Navigate to the official Splunk Enterprise Download Portal
- Select macOS platform and download the **DMG** installer
- Double-click the `.dmg` file to mount it

**Step 3 — Installation and Daemon Setup:**
- Run the Install Splunk wizard, accept the license, and install to the primary drive
- Create an admin username and password when the installer terminal prompts — this registers Splunk as a background daemon

**Step 4 — Web Console & Port Provisioning:**
1. Open `http://localhost:8000` in a browser
2. Log in with credentials from Step 3
3. Navigate to **Settings → Forwarding and receiving**
4. Under **Receive data**, click **Add new**, set listening port to **9997**, and save

---

### 3.2: Virtualization Setup & Windows Server Configuration (UTM)

Standard x86_64 hypervisors (VMware, VirtualBox) cannot run natively on Apple Silicon. **UTM** was used to provide hardware-accelerated virtualization for an ARM64 guest OS.

**Step 1 — Install UTM:**
- Download the UTM `.dmg` from the official repository
- Drag UTM into `/Applications`

**Step 2 — Provision the Windows VM:**
- Select **Virtualize** mode (not emulate) for native CPU performance
- Mount the **Windows ARM64** installation image (`.VHDX` or `.ISO`)
- Allocate **2 CPU cores** and **3 GB RAM**

**Step 3 — Complete Windows Setup:**
- Choose **Desktop Experience (GUI)** during installation
- Set up local Administrator credentials
- Install **UTM Guest Tools** for display drivers and clipboard sharing

**Step 4 — Network Configuration:**
- Set Network Mode to **Shared Network (NAT)**
- This allows the guest VM to reach the internet AND access the host's local IP for shipping telemetry

---

### 3.3: Sysmon Setup and Configuration

Microsoft Sysmon was installed inside the Windows VM to generate granular security event telemetry (process creation, registry modifications, network connections, etc.).

**Step 1 — Acquire Tools:**
- Download Sysmon binaries from [Microsoft Sysinternals](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon)
- Acquire a `sysmonconfig.xml` ruleset aligned with real-world threat indicators

**Step 2 — Install (ARM64 Native):**

Open PowerShell as Administrator:
```powershell
.\Sysmon64A.exe -i sysmonconfig.xml
```
> On M1-hosted VMs, the ARM64 binary `Sysmon64A.exe` must be used instead of the standard x64 version.

**Step 3 — Verify Logs:**
- Open **Event Viewer** (`eventvwr.msc`)
- Navigate to: `Applications and Services Logs → Microsoft → Windows → Sysmon → Operational`
- Confirm **Event ID 1 (Process Creation)** entries are actively populating

---

### 3.4: Splunk Universal Forwarder Installation on Windows (Guest VM)

The Splunk Universal Forwarder acts as the lightweight log-shipping agent, streaming Sysmon telemetry from the Windows VM to the macOS Splunk SIEM.

**Step 1 — Discover Host IP:**

Run on the macOS host terminal:
```bash
ipconfig getifaddr en0
```
Note the resulting IP (e.g., `192.168.85.102`) — this is the forwarding destination.

**Step 2 — Run the MSI Installer:**
1. Launch the 64-bit Splunk Universal Forwarder `.msi` installer inside the VM
2. Accept the license; select **An on-premises Splunk Enterprise instance**
3. Create local forwarder admin credentials
4. Skip the Deployment Server screen (not needed for single-node lab)
5. In **Receiving Indexer**, enter the macOS host IP and port **9997**
6. Click **Install** with administrative permissions

---

### 3.5: Configuring Forwarder Routing Inputs

After installation, the forwarder must be told which log streams to collect.

**Step 1 — Edit `inputs.conf`:**

Open with elevated PowerShell:
```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
```

Add the following:
```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
renderXml = true
index = main
sourcetype = XmlWinEventLog

[WinEventLog://Security]
disabled = 0
index = main
sourcetype = WinEventLog:Security
```

**Step 2 — Restart the Forwarder Service:**
```powershell
Restart-Service -Name SplunkForwarder
```

---

### 3.6: Invoke-AtomicRedTeam Framework Setup

The **Invoke-AtomicRedTeam (ART)** framework was installed to automate MITRE ATT&CK-aligned adversary emulation tests.

**Step 1 — Bypass Execution Policy:**
```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
```

**Step 2 — Install Framework:**
```powershell
# Pull installer from Red Canary's GitHub
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing)

# Install framework + all Atomics definitions
Install-AtomicRedTeam -InstallAtomics
```
Files install to `C:\AtomicRedTeam\`

**Step 3 — Verify Module Import:**
```powershell
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
```

---

## Splunk Dashboard Validation

Before emulation testing, a baseline verification confirmed the full data pipeline was functional.

- Opened `http://localhost:8000` and launched **Search & Reporting**
- Ran a broad discovery query over "All time":

```spl
index=* source="WinEventLog:Microsoft-Windows-Sysmon/Operational"
```

**Validation Results:**
- ✅ Active XML log rows arriving from the guest workstation
- ✅ Host field shows guest VM computer name
- ✅ Sourcetype = `WinEventLog:Microsoft-Windows-Sysmon/Operational`
- ✅ Event count climbing continuously — no dropped TCP packets on port 9997

---

## Emulation & Detection Matrix

---

### Technique 1: Scheduled Task/Job — T1053.005

| Field | Detail |
|-------|--------|
| **MITRE ID** | T1053.005 |
| **Adversary Intent** | Persistence & Privilege Escalation |
| **Key Event ID** | Event ID 1 (Process Creation) |

**Emulation:**
```powershell
Invoke-AtomicTest T1053.005
```

**Detection Query (Splunk):**
```spl
index="main" source="WinEventLog:Microsoft-Windows-Sysmon/Operational" "schtasks.exe" "create"
```

**Justification:** When an adversary creates a scheduled task via CLI, Windows invokes `schtasks.exe`. Event ID 1 captures its launch and exposes the full command-line footprint — the `/create` switch, task name (`/tn`), and payload path (`/tr`).

---

### Technique 2: Signed Binary Proxy Execution (Mshta) — T1218.005

| Field | Detail |
|-------|--------|
| **MITRE ID** | T1218.005 |
| **Adversary Intent** | Defense Evasion |
| **Key Event IDs** | Event ID 1 (Process Creation) & Event ID 3 (Network Connection) |

**Emulation:**
```powershell
Invoke-AtomicTest T1218.005
```

**Detection Query (Splunk):**
```spl
index=main source="WinEventLog:Microsoft-Windows-Sysmon/Operational" mshta vbscript
```

**Justification:** Event ID 1 catches `mshta.exe` executing outside its normal context with anomalous `.hta` file arguments. Event ID 3 confirms the outbound network connection made by the trusted Microsoft binary to pull down a remote payload.

---

### Technique 3: OS Credential Dumping (LSASS Memory) — T1003.001

| Field | Detail |
|-------|--------|
| **MITRE ID** | T1003.001 |
| **Adversary Intent** | Credential Access |
| **Key Event ID** | Event ID 10 (Process Access) |

**Emulation:**
```powershell
Invoke-AtomicTest T1003.001
```
> Executes `procdump` against `lsass.exe` to dump memory contents for credential harvesting.

**Detection Query (Splunk):**
```spl
index="main" sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational" lsass.exe
```

**Justification:** Adversaries can rename dump utilities to bypass name-based filters. Event ID 10 captures the exact moment any process opens a handle to `lsass.exe`'s memory space, logging the specific permission mask (`GrantedAccess=0x1fffff` or `0x1010`) — definitive forensic proof of a credential harvesting attempt.

---

### Technique 4: Command and Scripting Interpreter (PowerShell) — T1059.001

| Field | Detail |
|-------|--------|
| **MITRE ID** | T1059.001 |
| **Adversary Intent** | Execution |
| **Key Event ID** | Event ID 1 (Process Creation) |

**Emulation:**
```powershell
Invoke-AtomicTest T1059.001
```
> Executes an in-memory download and instant execution of a remotely hosted script.

**Detection Query (Splunk):**
```spl
index="main" sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational" powershell.exe (DownloadString OR Net.WebClient OR Invoke-WebRequest)
```

**Justification:** Fileless cradles execute entirely in volatile memory. Event ID 1 logs the initial plaintext argument block passed into PowerShell at runtime. Even if partially obfuscated, tracking `Net.WebClient`, `DownloadString`, or `Microsoft.XMLHTTP` in the `CommandLine` field flags anomalous script interpretation.

---

### Technique 5: Modify Registry — Disable Windows Defender (T1112)

| Field | Detail |
|-------|--------|
| **MITRE ID** | T1112 |
| **Adversary Intent** | Defense Evasion |
| **Key Event ID** | Event ID 13 (Registry Value Set) |

**Emulation:**
```powershell
Invoke-AtomicTest T1112
```
> Appends/modifies registry keys to disable real-time antivirus protection.

**Detection Query (Splunk):**
```spl
index="main" sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational" "T1112"
```

**Justification:** Adversaries use various parent methods to alter defenses, making process tracking alone insufficient. Event ID 13 records the actual registry modification committed to the hive database, identifying the exact `TargetObject` path (`...\Policies\Microsoft\Windows Defender`), the value mutated (`DisableAntiSpyware`), and the state change (`Details=DWORD (0x00000001)`).

---

## Summary

| # | Technique | MITRE ID | Intent | Key Event ID |
|---|-----------|----------|--------|--------------|
| 1 | Scheduled Task | T1053.005 | Persistence | Event ID 1 |
| 2 | Mshta Proxy Execution | T1218.005 | Defense Evasion | Event ID 1 & 3 |
| 3 | LSASS Credential Dump | T1003.001 | Credential Access | Event ID 10 |
| 4 | PowerShell Execution | T1059.001 | Execution | Event ID 1 |
| 5 | Registry Modification | T1112 | Defense Evasion | Event ID 13 |
