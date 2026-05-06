# Alternative A2DP Driver - Reverse Engineering Findings

**Date:** 2026-05-06  
**Target:** Alternative A2DP Driver v1.8.0.1 (Kernel Driver) / v1.8.2.1 (GUI/Service)  
**Vendor:** Luculent Systems, LLC  
**Install Path:** `C:\Program Files\Luculent Systems\AltA2DP\`

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Component Breakdown](#2-component-breakdown)
3. [Kernel Driver Deep Dive](#3-kernel-driver-deep-dive)
4. [Registry Schema (Complete)](#4-registry-schema-complete)
5. [Codec Parameter Reference](#5-codec-parameter-reference)
6. [Licensing System](#6-licensing-system)
7. [Driver Switching Mechanism](#7-driver-switching-mechanism)
8. [Building a Custom GUI](#8-building-a-custom-gui)
9. [Useful Commands](#9-useful-commands)

---

## 1. Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│  AltA2dpConfig.exe  (WinUI 3 XAML GUI)                   │
│  - Per-device codec configuration                        │
│  - License verification (Ed25519)                        │
│  - Writes codec params to Registry\Next\                 │
│  - Launches AltA2dpDriverInstaller.exe for driver swap   │
└────────────────────────┬─────────────────────────────────┘
                         │ Registry writes + CreateProcessW
┌────────────────────────┼─────────────────────────────────┐
│  AltA2dpSVC.exe        │  (Win32 Service)                │
│  - PnP device monitoring (CM_Register_Notification)      │
│  - Device enable/disable cycling (CM_Enable/Disable)     │
│  - Registry change watcher (RegNotifyChangeKeyValue)     │
│  - AVRCP coordination (BTHENUM\{0000110E-...})           │
│  - Bridge between Config GUI and kernel driver            │
└────────────────────────┼─────────────────────────────────┘
                         │ Registry reads
┌────────────────────────┼─────────────────────────────────┐
│  AltA2DP.sys           │  (Kernel Driver - PortCls/KMDF) │
│  - NO license checks                                    │
│  - Reads codec config from Registry                      │
│  - Audio encoding (SBC/AAC/aptX/aptX HD/aptX LL/LDAC)   │
│  - AVDTP protocol handling                               │
│  - L2CAP transport via Windows BT stack                  │
└────────────────────────┼─────────────────────────────────┘
                         │ Bluetooth L2CAP
                         ▼
                   Headphones / Speaker
```

**Key Insight:** The kernel driver has **zero license enforcement**. All licensing is in the GUI. The driver reads whatever codec configuration is in the registry and uses it unconditionally.

---

## 2. Component Breakdown

### Files

| File | Size | Type | Purpose |
|------|------|------|---------|
| `AltA2DP.sys` | 818 KB | Kernel driver (KMDF + PortCls) | Audio encoding + BT transport |
| `AltA2dpSVC.exe` | 75 KB | Win32 service | PnP lifecycle + config bridge |
| `AltA2dpConfig.exe` | 3 MB | WinUI 3 XAML app | GUI + licensing |
| `AltA2dpDriverInstaller.exe` | 130 KB | CLI tool | Driver install/uninstall |
| `AltA2DP.inf` | 14 KB | Driver INF (UTF-16LE) | PnP registration |
| `AltA2DP.cat` | 12 KB | Catalog file | WHQL signature |

### PE Details (AltA2DP.sys)

- **Machine:** x86_64 (AMD64)
- **Subsystem:** NATIVE (kernel mode)
- **Framework:** KMDF (WDF 1.27) + PortCls audio miniport
- **PDB Path:** `C:\Products\AltA2DP\AltA2DP-1.8.0-f\DRV\Build\x64\Release\AltA2DP.pdb`
- **Sections:** `.text` (604KB), `.rdata` (155KB), `.data` (7KB), `PAGE` (14KB), `INIT` (3KB)

---

## 3. Kernel Driver Deep Dive

### Framework

The driver is a **PortCls Audio Miniport Driver** — the same framework used by sound card drivers. It implements two subdevices:

1. **"Alternative A2DP Topology"** — mixer/volume controls, audio endpoint properties
2. **"Alternative A2DP Wave"** — PCM audio rendering (receives audio from Windows audio engine)

### Key Imports

**From `portcls.sys`:**
- `PcInitializeAdapterDriver` — DriverEntry registration
- `PcAddAdapterDevice` — device creation on PnP discovery
- `PcNewPort` / `PcRegisterSubdevice` — creates Wave + Topology miniports
- `PcRegisterPhysicalConnection` — wires Topology to Wave
- `PcDispatchIrp` — IRP routing through PortCls
- `PcNewServiceGroup` — DPC scheduling
- `PcRegisterAdapterPowerManagement` — power management

**From `ntoskrnl.exe`:**
- `KeSaveExtendedProcessorState` / `KeRestoreExtendedProcessorState` — SSE/AVX SIMD for audio encoding
- `PsCreateSystemThread` — worker threads for encoding/transport
- `ExAllocateTimer` / `ExSetTimer` — packet scheduling
- `IoRegisterPlugPlayNotification` — PnP device monitoring
- `IoRegisterDeviceInterface` — exposes KS audio interfaces

**From `WDFLDR.SYS`:**
- `WdfVersionBind` / `WdfVersionBindClass` — KMDF framework binding

### Hardware ID

```
BTHENUM\{0000110b-0000-1000-8000-00805f9b34fb}
```

This is the **Bluetooth A2DP Audio Sink** service class UUID. By registering for this hardware ID, the driver replaces Microsoft's built-in `BthA2dp.sys`.

### KS Interface Categories

- `KSCATEGORY_AUDIO` — `{6994ad04-93ef-11d0-a3cc-00a0c9223196}`
- `KSCATEGORY_RENDER` — `{65e8773e-8f56-11d0-a3b9-00a0c9223196}`
- `KSCATEGORY_TOPOLOGY` — `{dda54a40-1e4c-11d1-a050-405705c10000}`

### Codecs (all compiled into the .sys)

| Codec | Evidence | Notes |
|-------|----------|-------|
| **SBC** | `SbcChannelMode`, `SbcBitpool`, etc. | Mandatory A2DP baseline, open standard |
| **AAC** | `"AAC Encoder"`, `"SBR Encoder"`, `"MPEG Surround Encoder"` | Fraunhofer FDK AAC (statically linked) |
| **aptX** | `AptxChannelMode`, `AptxSamplingFrequency` | Qualcomm proprietary |
| **aptX HD** | `AptxHdSampleFormat` (24-bit support) | High-definition variant |
| **aptX LL** | `AptxLlChannelMode`, `LlLatency`, `LlBufferLevel` | Low-latency variant |
| **LDAC** | `LdacChannelMode`, `LdacEqmid`, `LdacSampleFormat` | Sony hi-res codec |

Audio encoding happens in **kernel mode** using **SSE/AVX SIMD** instructions (confirmed by extended processor state save/restore imports).

### Bluetooth Protocol

The driver implements **AVDTP** (Audio/Video Distribution Transport Protocol) directly:
- `UseAvdtpReconfigureCmd` — codec switching on-the-fly
- `DiscoverDelayIn/Out`, `SnkDiscoverTO` — service discovery timing
- Transmit buffer management: `TxBufferCriticalThreshold`, `TxBufferHighThreshold`, `TxBufferLowThreshold`, `TxFlushTO`, `TxLatency`, `TxMaxDuration`, `TxMaxSize`, `TxPeriod`, `TxSendAhead`

### Adaptive Bitrate (ABR)

Built-in link quality monitoring:
- `AbrEnable` — toggle
- `AbrShortTermHighThreshold` / `AbrShortTermLowThreshold` — immediate response
- `AbrLongTermHighPct/Threshold`, `AbrLongTermLowPct/Threshold` — sustained trends
- `AbrBufferHighShortTermAction` — action on buffer overflow
- `AbrLongTermPeriod` — measurement window

### Data Flow

```
Windows Audio Engine (audiodg.exe)
       │ PCM audio via WaveRT/KS pin
       ▼
┌──────────────────────┐
│   AltA2DP.sys         │  PortCls Wave Miniport
│                       │
│  1. Receive PCM       │
│  2. Encode via        │  ◄── SSE/AVX SIMD optimized
│     SBC/AAC/aptX/LDAC │
│  3. AVDTP framing     │  ◄── RTP headers + media packets
│  4. L2CAP transmit    │  ◄── via Windows BT stack
└───────────┬───────────┘
            │ Bluetooth L2CAP
            ▼
  Headphones/Speaker (A2DP Sink)
```

---

## 4. Registry Schema (Complete)

### Base Path

```
HKLM:\SYSTEM\CurrentControlSet\Services\AltA2DP\Parameters\
```

### Structure

```
Parameters\
├── LogPages          (DWORD) = 16        ← WPP tracing log pages
├── VerboseOn         (DWORD) = 1         ← verbose logging
├── DriverVersion     (DWORD) = 17301505  ← packed version
├── Wdf\
│   ├── WdfMajorVersion = 1
│   └── WdfMinorVersion = 27
└── Devices\
    ├── Capability\         ← What the remote device supports (read-only, populated by driver)
    │   ├── {bt_address}\   ← e.g., 0000340e224a88bb
    │   │   ├── Name                    (string)  ← Device friendly name
    │   │   ├── Codec                   (DWORD)   ← Bitmask of supported codecs
    │   │   ├── SbcChannelMode          (DWORD)
    │   │   ├── SbcSamplingFrequency    (DWORD)
    │   │   ├── SbcAllocationMethod     (DWORD)
    │   │   ├── SbcSubbands             (DWORD)
    │   │   ├── SbcBlockLength          (DWORD)
    │   │   ├── SbcMinimumBitpool       (DWORD)
    │   │   ├── SbcMaximumBitpool       (DWORD)
    │   │   ├── AbrEnable               (DWORD)
    │   │   ├── AacChannelMode          (DWORD)
    │   │   ├── AacSamplingFrequency    (DWORD)
    │   │   ├── AacBitrate              (DWORD)
    │   │   ├── AacPeakBitrate          (DWORD)
    │   │   ├── LdacChannelMode         (DWORD)
    │   │   ├── LdacSamplingFrequency   (DWORD)
    │   │   ├── LdacSampleFormat        (DWORD)
    │   │   ├── LdacEqmid               (DWORD)
    │   │   ├── AptxChannelMode         (DWORD)
    │   │   ├── AptxSamplingFrequency   (DWORD)
    │   │   ├── AptxHdChannelMode       (DWORD)
    │   │   ├── AptxHdSamplingFrequency (DWORD)
    │   │   ├── AptxHdSampleFormat      (DWORD)
    │   │   ├── AptxLlChannelMode       (DWORD)
    │   │   └── AptxLlSamplingFrequency (DWORD)
    │   └── {bt_address_2}\
    │       └── ...
    ├── Current\            ← What is actively negotiated (read-only, updated by driver in real-time)
    │   └── {bt_address}\
    │       ├── (all same codec fields as Capability)
    │       ├── *Mask fields           ← Bitmask versions of each negotiated param
    │       ├── Opened          (DWORD) ← 1 = connected, 0 = disconnected
    │       ├── Bitrate         (DWORD) ← actual bitrate in bps
    │       ├── ScoActive       (DWORD) ← 1 = HFP call active, A2DP suspended
    │       ├── Delay           (DWORD) ← transport delay in 100ns units
    │       └── Error           (DWORD) ← error code (0 = none)
    └── Next\               ← What to use on next connection (WRITABLE - this is what the GUI writes)
        └── {bt_address}\
            ├── (all same codec fields as Capability)
            └── VolumeLevel     (DWORD) ← volume (signed 32-bit, negative = attenuation)
```

### Device Address Format

The `{bt_address}` key name is the Bluetooth address zero-padded to 16 hex chars:
```
Actual BT MAC: 34:0E:22:4A:88:BB
Registry key:  0000340e224a88bb
```

You can extract the BT address from the PnP device ID:
```
BTHENUM\{0000110b-...}_VID&0001004c_PID&2027\9&3a3a251&0&340E224A88BB_C00000000
                                                         ^^^^^^^^^^^^
                                                         BT address here
```

---

## 5. Codec Parameter Reference

### Codec Type Values

| Value | Codec | Notes |
|-------|-------|-------|
| 0 | None | Not supported / not configured |
| 1 | SBC | Mandatory baseline codec |
| 2 | AAC | Advanced Audio Coding |
| 3 | SBC + AAC | Device supports both (Capability only) |

> **Note:** aptX/LDAC codec type values were not observed in active use (Apple devices don't support them). They are likely higher values (4, 5, 6, etc.) or encoded differently.

### SBC Parameters

| Parameter | Bit Values | Meaning |
|-----------|-----------|---------|
| **SbcChannelMode** | 1 = Joint Stereo | 2 = Stereo | 4 = Dual Channel | 8 = Mono | 15 = All |
| **SbcSamplingFrequency** | 1 = 48 kHz | 2 = 44.1 kHz | 4 = 32 kHz | 8 = 16 kHz |
| **SbcAllocationMethod** | 1 = Loudness | 2 = SNR | 3 = Both |
| **SbcSubbands** | 1 = 8 | 2 = 4 | 3 = Both |
| **SbcBlockLength** | 1 = 16 | 2 = 12 | 4 = 8 | 8 = 4 | 15 = All |
| **SbcMinimumBitpool** | 2-250 | Minimum bitpool value |
| **SbcMaximumBitpool** | 2-250 | Maximum bitpool value (53 = standard max) |
| **AbrEnable** | 0 = off | 1 = on | Adaptive bitrate control |

### AAC Parameters

| Parameter | Bit Values | Meaning |
|-----------|-----------|---------|
| **AacChannelMode** | 4 = Stereo | 8 = Mono | 12 = Both |
| **AacSamplingFrequency** | 8 = 44.1 kHz | 16 = 48 kHz | 24 = Both |
| **AacBitrate** | Integer | Target bitrate in bps (e.g., 256000) |
| **AacPeakBitrate** | Integer | Peak/max bitrate in bps (e.g., 376875) |

> For Capability keys, values are bitmasks (OR'd together) of what the device supports.  
> For Current/Next keys, values are single selections.

### LDAC Parameters

| Parameter | Bit Values | Meaning |
|-----------|-----------|---------|
| **LdacChannelMode** | 0 = N/A | 1 = Stereo | 2 = Dual Channel | 4 = Mono |
| **LdacSamplingFrequency** | 1 = 44.1 kHz | 2 = 48 kHz | 4 = 88.2 kHz | 8 = 96 kHz |
| **LdacSampleFormat** | 1 = 16-bit | 2 = 24-bit | 4 = 32-bit |
| **LdacEqmid** | 0 = Quality (990 kbps) | 1 = Normal (660 kbps) | 2 = Connection (330 kbps) | 3 = ABR |

### aptX Parameters

| Parameter | Bit Values | Meaning |
|-----------|-----------|---------|
| **AptxChannelMode** | 0 = N/A | 1 = Mono | 2 = Stereo |
| **AptxSamplingFrequency** | 0 = N/A | 1 = 44.1 kHz | 2 = 48 kHz |

### aptX HD Parameters

| Parameter | Bit Values | Meaning |
|-----------|-----------|---------|
| **AptxHdChannelMode** | 0 = N/A | 1 = Mono | 2 = Stereo |
| **AptxHdSamplingFrequency** | 0 = N/A | 1 = 44.1 kHz | 2 = 48 kHz |
| **AptxHdSampleFormat** | 0 = N/A | 1 = 16-bit | 2 = 24-bit |

### aptX Low Latency Parameters

| Parameter | Bit Values | Meaning |
|-----------|-----------|---------|
| **AptxLlChannelMode** | 0 = N/A | 1 = Mono | 2 = Stereo |
| **AptxLlSamplingFrequency** | 0 = N/A | 1 = 44.1 kHz | 2 = 48 kHz |

---

## 6. Licensing System

### Overview

- **Crypto:** Ed25519 digital signatures (64 bytes / 512 bits)
- **Verification:** Asymmetric — public key embedded in EXE, private key on server
- **Storage:** Registry blobs at `HKLM:\Software\Luculent Systems\Alternative A2DP Driver\License`
- **File format:** `.aalic` files (plain text key-value pairs + signature)
- **Enforcement:** GUI-only. The kernel driver has NO license checks.

### License Types

| Type | Registry Value | Duration | Fingerprint |
|------|---------------|----------|-------------|
| Trial | `TrialLicense` | 7 days | BT adapter MAC hash + OS install ID hash (2 values) |
| Paid | `PaidLicense` | Permanent | Full hardware fingerprint (motherboard, CPU, NICs, serial hashes, etc.) |

### Trial License Format

```
licensed_software:Alternative A2DP Driver (Trial)
version:01080201
local_address_hash:{32-bit hash of BT adapter MAC}
os_id_hash:{32-bit hash of Windows installation ID}
start:{unix timestamp}
expire:{unix timestamp, start + 7 days}
signature:{128 hex chars = 64 bytes Ed25519}
```

### Paid License Format

```
licensed_software:Alternative A2DP Driver with AAC CODEC Version 1.x
system_manufacturer:{SMBIOS manufacturer}
system_product_name:{SMBIOS product}
system_version:{SMBIOS version}
system_sku_number:{SMBIOS SKU}
system_family:{SMBIOS family}
system_serial_number_hash:{32-bit hash}
system_uuid_hash:{32-bit hash}
processor_version:{CPU model string}
local_address_hash:{32-bit hash of BT MAC}
nic1:{MAC address}
nic1_name:{NIC friendly name}
nic1_id:{NIC PCI device ID}
... (up to nic3)
lpc_name:{LPC bridge name}
lpc_id:{LPC bridge PCI ID}
os_id_hash:{32-bit hash}
license_sku:{tier code, e.g., AC1d}
license_type:single_system
licensee:{email}
licensed_date:{RFC 2822 date}
signature:{128 hex chars Ed25519}
```

### License Tiers

| SKU | What it unlocks |
|-----|----------------|
| (none) | SBC only |
| `AC1d` | SBC + AAC |
| (unknown) | Higher tiers for aptX, LDAC presumably exist |

### Hardware Fingerprint Sources

The Config GUI uses these WinRT APIs:
- `Windows.System.Profile.HardwareIdentification`
- `Windows.System.Profile.SystemIdentification`
- `Windows.Security.Cryptography.Core.AsymmetricKeyAlgorithmProvider` (Ed25519 verify)
- `Windows.Security.Cryptography.Core.HashAlgorithmProvider` (hashing)
- WMI `root\wmi` (SMBIOS data)

### License Verification Flow

```
Trial:
  1. User clicks "Fetch Trial License"
  2. App collects local_address_hash + os_id_hash
  3. HTTP POST to Luculent backend (WinRT HttpClient)
  4. Server checks if these hashes have trialed before
  5. If new: returns signed 7-day trial license
  6. If seen: returns error → "NoTrialLicense" / "NoMoreAacTrialMessage"
  7. App verifies Ed25519 signature, stores in registry

Purchase:
  1. User buys on product website
  2. Receives .aalic file
  3. In Config app: "Apply Purchased License" → FileOpenPicker (*.aalic)
  4. App verifies Ed25519 signature
  5. App compares hardware fingerprint in license vs current machine
  6. Match → stored as PaidLicense, codec unlocked
  7. Mismatch → "LicenseFileMismatch" error
```

### Local Persistence

The ONLY local storage for licensing is:
```
HKLM:\Software\Luculent Systems\Alternative A2DP Driver\License\TrialLicense  (REG_BINARY)
HKLM:\Software\Luculent Systems\Alternative A2DP Driver\License\PaidLicense   (REG_BINARY)
```

No files on disk. No AppData. No ProgramData. Server-side does the "already trialed" tracking.

---

## 7. Driver Switching Mechanism

### How AltA2DP replaces Microsoft's driver

Both drivers register for the same hardware ID:
```
BTHENUM\{0000110b-0000-1000-8000-00805f9b34fb}   (A2DP Audio Sink UUID)
```

Windows PnP selects the highest-ranked driver. AltA2DP's 3rd-party INF wins over the inbox `BthA2dp.sys`.

### Install flow (AltA2dpDriverInstaller.exe)

Key API calls:
1. `SetupCopyOEMInfW("Driver\AltA2DP.inf")` — copies INF to driver store
2. `SetupDiGetClassDevsW` + `SetupDiEnumDeviceInfo` — finds all BTHENUM A2DP devices
3. `DiInstallDevice` (newdev.dll) — re-evaluates driver selection per device node
4. Device restarts with `AltA2DP.sys`

### Uninstall / switch back to Windows driver

```powershell
# Find the OEM INF name
pnputil /enum-drivers | Select-String "alta2dp" -Context 0,8

# Remove from driver store (forces fallback to inbox BthA2dp.sys)
pnputil /delete-driver oem##.inf /uninstall
```

Or use Device Manager → Update Driver → "Browse my computer" → "Let me pick" → select "Microsoft Bluetooth A2dp driver".

### Per-device switching

Each BT device is a separate PnP device node:
```
BTHENUM\{0000110b-...}_VID&0001004c_PID&2027\..._340E224A88BB_C00000000  ← AirPods Pro
BTHENUM\{0000110b-...}_VID&0001004c_PID&201f\..._70F94AA804E3_C00000000  ← AirPods Max
```

`DiInstallDevice` can target individual device nodes, allowing different drivers per device.

---

## 8. Building a Custom GUI

### Prerequisites

- AltA2DP driver already installed (via the official installer)
- AltA2dpSVC.exe service running (handles PnP lifecycle)

### What your GUI needs to do

| Function | API / Method |
|----------|-------------|
| **List devices** | Enumerate `HKLM:\...\Devices\Capability\*` subkeys, or query `Win32_PnPEntity` where `DeviceID LIKE '%BTHENUM%110b%'` |
| **Read capabilities** | `Get-ItemProperty "HKLM:\...\Devices\Capability\{addr}"` |
| **Show live status** | `Get-ItemProperty "HKLM:\...\Devices\Current\{addr}"` — poll or use `RegNotifyChangeKeyValue` for push notifications |
| **Change codec** | `Set-ItemProperty "HKLM:\...\Devices\Next\{addr}" -Name "Codec" -Value 2` (plus codec-specific params) |
| **Apply changes** | `Disable-PnpDevice` then `Enable-PnpDevice` on the BTHENUM instance ID, or call `CM_Disable_DevNode`/`CM_Enable_DevNode` via P/Invoke |
| **Detect new devices** | `CM_Register_Notification` on BTHENUM device interface, or WMI `__InstanceCreationEvent` watcher |
| **Switch drivers** | `pnputil /delete-driver` + `pnputil /add-driver` or `DiInstallDevice` via P/Invoke |

### Technology choices

| Stack | Pros | Cons |
|-------|------|------|
| **PowerShell script** | Zero dependencies, fast to build | No real-time updates, ugly |
| **Electron / Tauri** | Modern UI, cross-platform potential | Needs native bridge for registry + PnP |
| **C# WPF / WinUI 3** | Native Windows, direct registry + P/Invoke | Heavier tooling |
| **Python + tkinter/Qt** | Fast prototyping | Needs `winreg` + `ctypes` for PnP APIs |
| **C# MAUI** | Modern, cross-platform | Overkill for this |

### Recommended: Electron + PowerShell backend

```
Electron app
├── Frontend (HTML/CSS/JS)
│   ├── Device list (sidebar)
│   ├── Codec selector (dropdown)
│   ├── Parameter sliders/checkboxes
│   └── Live status display
└── Backend (Node.js child_process → PowerShell)
    ├── Read registry for device/codec state
    ├── Write registry for codec changes
    └── PnP device cycling for reconnect
```

### Minimum viable script to read all devices

```powershell
$base = "HKLM:\SYSTEM\CurrentControlSet\Services\AltA2DP\Parameters\Devices"
$codecMap = @{0='None'; 1='SBC'; 2='AAC'; 3='SBC+AAC'}

Get-ChildItem "$base\Capability" -EA SilentlyContinue | ForEach-Object {
    $addr = $_.PSChildName
    $cap = Get-ItemProperty $_.PSPath
    $cur = Get-ItemProperty "$base\Current\$addr" -EA SilentlyContinue
    $nxt = Get-ItemProperty "$base\Next\$addr" -EA SilentlyContinue

    [PSCustomObject]@{
        Device       = $cap.Name
        Address      = $addr
        Supports     = $codecMap[[int]$cap.Codec]
        CurrentCodec = if ($cur) { $codecMap[[int]$cur.Codec] } else { "N/A" }
        Connected    = if ($cur) { $cur.Opened -eq 1 } else { $false }
        Bitrate      = if ($cur -and $cur.Bitrate -gt 0) { "$([math]::Round($cur.Bitrate/1000))kbps" } else { "-" }
        NextCodec    = if ($nxt) { $codecMap[[int]$nxt.Codec] } else { "N/A" }
    }
} | Format-Table -AutoSize
```

### Switch a device to AAC

```powershell
$addr = "0000340e224a88bb"  # Your AirPods Pro BT address
$nextPath = "HKLM:\SYSTEM\CurrentControlSet\Services\AltA2DP\Parameters\Devices\Next\$addr"

# Write AAC codec parameters
Set-ItemProperty $nextPath -Name "Codec" -Value 2
Set-ItemProperty $nextPath -Name "AacChannelMode" -Value 4          # Stereo
Set-ItemProperty $nextPath -Name "AacSamplingFrequency" -Value 8    # 44.1 kHz
Set-ItemProperty $nextPath -Name "AacBitrate" -Value 256000         # 256 kbps

# Force reconnect to apply (requires admin)
$instanceId = "BTHENUM\{0000110B-0000-1000-8000-00805F9B34FB}_VID&0001004C_PID&2027\9&3A3A251&0&340E224A88BB_C00000000"
Disable-PnpDevice -InstanceId $instanceId -Confirm:$false
Start-Sleep -Seconds 2
Enable-PnpDevice -InstanceId $instanceId -Confirm:$false
```

### Watch for live status changes

```powershell
# Poll-based (simple)
while ($true) {
    $cur = Get-ItemProperty "HKLM:\...\Devices\Current\0000340e224a88bb" -EA SilentlyContinue
    if ($cur) {
        $codec = @{0='None';1='SBC';2='AAC'}[[int]$cur.Codec]
        $status = if ($cur.Opened -eq 1) { "Connected" } else { "Disconnected" }
        Write-Host "[$([DateTime]::Now.ToString('HH:mm:ss'))] $status | $codec | $([math]::Round($cur.Bitrate/1000))kbps | Delay: $($cur.Delay * 0.1)ms"
    }
    Start-Sleep -Seconds 2
}
```

---

## 9. Useful Commands

### Device Enumeration

```powershell
# List all A2DP devices managed by AltA2DP
Get-CimInstance Win32_PnPEntity |
    Where-Object { $_.DeviceID -like "*BTHENUM*110b*" } |
    Select-Object DeviceID, Name, Status, Manufacturer

# Check which driver is active
pnputil /enum-drivers | Select-String "alta2dp|btha2dp" -Context 0,8

# List BTHENUM device instances
pnputil /enum-devices /class Media |
    Select-String "a2dp|BTHENUM|alternative" -Context 0,5
```

### Service Management

```powershell
# Check driver and service status
Get-Service AltA2DP       # Kernel driver
Get-Service AltA2dpSVC    # User-mode service

# Restart the service (if config changes aren't being picked up)
Restart-Service AltA2dpSVC
```

### Registry Quick Access

```powershell
# Dump all devices and their current codec
$base = "HKLM:\SYSTEM\CurrentControlSet\Services\AltA2DP\Parameters\Devices"
"Capability","Current","Next" | ForEach-Object {
    $state = $_
    Get-ChildItem "$base\$state" -EA SilentlyContinue | ForEach-Object {
        $p = Get-ItemProperty $_.PSPath
        Write-Host "$state/$($_.PSChildName): Codec=$($p.Codec) Name=$($p.Name)"
    }
}

# Read license info
$lic = [System.Text.Encoding]::ASCII.GetString(
    (Get-ItemProperty "HKLM:\Software\Luculent Systems\Alternative A2DP Driver\License").PaidLicense
)
Write-Host $lic
```

### Driver Store Management

```powershell
# List the AltA2DP driver in the store
pnputil /enum-drivers | Select-String "alta2dp" -Context 0,8

# Install driver (if you have the .inf)
pnputil /add-driver "C:\path\to\AltA2DP.inf" /install

# Remove driver (reverts all devices to Windows BthA2dp.sys)
pnputil /delete-driver oem##.inf /uninstall

# Force a specific device to re-evaluate driver selection
pnputil /scan-devices
```

### ETW Tracing (driver debug logs)

```powershell
# The driver registers WPP tracing with this GUID:
# {8225dde8-3b11-49f2-93c0-83d9060351a4}

# Start a trace session
logman create trace AltA2DPTrace -p "{8225dde8-3b11-49f2-93c0-83d9060351a4}" -o alta2dp.etl
logman start AltA2DPTrace

# ... reproduce the issue ...

logman stop AltA2DPTrace
# Decode with: tracefmt alta2dp.etl -o output.txt (requires TMF/PDB)
```

---

## Legal Note

The `AltA2DP.sys` kernel driver is copyrighted by Luculent Systems, LLC and WHQL-signed under their certificate. Building a custom GUI for personal use on a machine where the driver is already installed and licensed is fine. Redistributing the driver binary (`.sys`, `.inf`, `.cat`) without authorization from Luculent Systems is copyright infringement.
