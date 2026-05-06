# Build Prompt: Custom GUI for Alternative A2DP Bluetooth Driver

Paste everything below this line into a new Claude / ChatGPT conversation.

---

## Context

I have a Windows kernel driver called "Alternative A2DP Driver" (by Luculent Systems) installed on my Windows 11 machine. It replaces Microsoft's built-in Bluetooth A2DP audio driver and adds support for high-quality audio codecs (AAC, aptX, aptX HD, aptX LL, LDAC) over Bluetooth. The driver is already installed, signed, and working.

The driver itself has NO license checks — it reads codec configuration from the Windows registry and encodes/streams whatever it's told to. The official GUI app just reads/writes registry values. I want to build my own custom GUI that does the same thing but with a better, more modern UI/UX.

## What I Need You to Build

A modern desktop app (Electron + React, or Tauri + React, or whatever you think looks best) that acts as a control panel for this Bluetooth audio driver. It should:

### Core Features

1. **Device List (sidebar)** — Show all connected Bluetooth A2DP devices with their name, connection status (connected/disconnected), and currently active codec. Auto-refresh when devices connect/disconnect.

2. **Codec Selector** — Per-device dropdown to choose: SBC, AAC, LDAC, aptX, aptX HD, aptX LL. Only show codecs the device actually supports (read from Capability registry key — if a codec's ChannelMode is 0, the device doesn't support it).

3. **Codec Parameters** — Per-codec settings panel that shows relevant options:
   - **SBC:** Channel Mode, Sampling Frequency, Allocation Method, Subbands, Block Length, Bitpool (min/max slider), ABR toggle
   - **AAC:** Channel Mode, Sampling Frequency, Bitrate slider (up to device's max)
   - **LDAC:** Channel Mode, Sampling Frequency, Sample Format (16/24/32-bit), Quality mode (Quality 990kbps / Normal 660kbps / Connection 330kbps / ABR)
   - **aptX:** Channel Mode, Sampling Frequency
   - **aptX HD:** Channel Mode, Sampling Frequency, Sample Format
   - **aptX LL:** Channel Mode, Sampling Frequency

4. **Live Status** — Real-time display of: current codec, bitrate, connection state, latency/delay, whether a phone call (SCO) is active.

5. **Apply / Reconnect Button** — Writes settings to the "Next" registry key and cycles the device (disable then re-enable) to force the driver to reconnect with new settings.

### Registry Schema

The driver stores everything at:
```
HKLM:\SYSTEM\CurrentControlSet\Services\AltA2DP\Parameters\Devices\
```

Under that path there are 3 subkeys, each containing per-device subkeys named by BT address:

- **`Capability\{bt_address}\`** — READ ONLY. What the remote device supports. Populated by the driver after first connection. Values are bitmasks of supported options.
- **`Current\{bt_address}\`** — READ ONLY. What is actively negotiated right now. Updated by the driver in real-time.
- **`Next\{bt_address}\`** — WRITABLE. What the driver should use on the next connection. **This is what we write to.**

The `{bt_address}` is a 16-char hex string like `0000340e224a88bb`.

### Registry Values Per Device

Every device subkey (Capability/Current/Next) has these DWORD values:

```
Codec                   — Which codec: 1=SBC, 2=AAC, 3=SBC+AAC (capability only)
SbcChannelMode          — 1=Joint Stereo, 2=Stereo, 4=Dual Channel, 8=Mono, 15=All
SbcSamplingFrequency    — 1=48kHz, 2=44.1kHz, 4=32kHz, 8=16kHz
SbcAllocationMethod     — 1=Loudness, 2=SNR, 3=Both
SbcSubbands             — 1=8, 2=4, 3=Both
SbcBlockLength          — 1=16, 2=12, 4=8, 8=4, 15=All
SbcMinimumBitpool       — 2-250
SbcMaximumBitpool       — 2-250 (53 = standard max)
AbrEnable               — 0=off, 1=on

AacChannelMode          — 4=Stereo, 8=Mono, 12=Both
AacSamplingFrequency    — 8=44.1kHz, 16=48kHz, 24=Both
AacBitrate              — Target bitrate in bps (e.g., 256000)
AacPeakBitrate          — Peak bitrate in bps

LdacChannelMode         — 0=N/A, 1=Stereo, 2=Dual Channel, 4=Mono
LdacSamplingFrequency   — 1=44.1kHz, 2=48kHz, 4=88.2kHz, 8=96kHz
LdacSampleFormat        — 1=16-bit, 2=24-bit, 4=32-bit
LdacEqmid               — 0=Quality(990kbps), 1=Normal(660kbps), 2=Connection(330kbps), 3=ABR

AptxChannelMode         — 0=N/A, 1=Mono, 2=Stereo
AptxSamplingFrequency   — 0=N/A, 1=44.1kHz, 2=48kHz

AptxHdChannelMode       — 0=N/A, 1=Mono, 2=Stereo
AptxHdSamplingFrequency — 0=N/A, 1=44.1kHz, 2=48kHz
AptxHdSampleFormat      — 0=N/A, 1=16-bit, 2=24-bit

AptxLlChannelMode       — 0=N/A, 1=Mono, 2=Stereo
AptxLlSamplingFrequency — 0=N/A, 1=44.1kHz, 2=48kHz
```

The `Current` key also has these extra fields:
```
Opened      — 1=connected, 0=disconnected
Bitrate     — actual bitrate in bps
ScoActive   — 1=phone call active (A2DP suspended), 0=no
Delay       — transport delay in 100ns units (divide by 10 for ms)
Error       — error code, 0=none
```

The `Next` key also has:
```
VolumeLevel — DWORD (signed 32-bit, negative = attenuation)
```

### How to Read Registry from the App

If using Electron/Node.js, you can shell out to PowerShell:

```javascript
const { execSync } = require('child_process');

function getDevices() {
  const script = `
    $base = "HKLM:\\SYSTEM\\CurrentControlSet\\Services\\AltA2DP\\Parameters\\Devices"
    $devices = @()
    Get-ChildItem "$base\\Capability" -EA SilentlyContinue | ForEach-Object {
      $addr = $_.PSChildName
      $cap = Get-ItemProperty $_.PSPath
      $cur = Get-ItemProperty "$base\\Current\\$addr" -EA SilentlyContinue
      $nxt = Get-ItemProperty "$base\\Next\\$addr" -EA SilentlyContinue
      $devices += @{
        name = $cap.Name
        address = $addr
        capability = @{
          codec = $cap.Codec
          sbcChannelMode = $cap.SbcChannelMode
          aacChannelMode = $cap.AacChannelMode
          aacBitrate = $cap.AacBitrate
          ldacChannelMode = $cap.LdacChannelMode
          aptxChannelMode = $cap.AptxChannelMode
          aptxHdChannelMode = $cap.AptxHdChannelMode
          aptxLlChannelMode = $cap.AptxLlChannelMode
          # ... include ALL fields
        }
        current = if ($cur) { @{
          codec = $cur.Codec; opened = $cur.Opened; bitrate = $cur.Bitrate
          scoActive = $cur.ScoActive; delay = $cur.Delay; error = $cur.Error
        }} else { $null }
        next = if ($nxt) { @{ codec = $nxt.Codec } } else { $null }
      }
    }
    $devices | ConvertTo-Json -Depth 5
  `;
  return JSON.parse(execSync(`powershell -Command "${script}"`, { encoding: 'utf-8' }));
}
```

### How to Write Settings

```javascript
function setCodec(address, codec, params) {
  // codec: 1=SBC, 2=AAC, etc.
  // params: object with codec-specific values
  const path = `HKLM:\\SYSTEM\\CurrentControlSet\\Services\\AltA2DP\\Parameters\\Devices\\Next\\${address}`;
  let script = `Set-ItemProperty "${path}" -Name "Codec" -Value ${codec}\n`;
  for (const [key, value] of Object.entries(params)) {
    script += `Set-ItemProperty "${path}" -Name "${key}" -Value ${value}\n`;
  }
  execSync(`powershell -Command "${script}"`, { encoding: 'utf-8' });
}
```

### How to Force Reconnect (Apply Settings)

```javascript
function reconnectDevice(instanceId) {
  // instanceId example: "BTHENUM\\{0000110B-0000-1000-8000-00805F9B34FB}_VID&0001004C_PID&2027\\9&3A3A251&0&340E224A88BB_C00000000"
  const script = `
    Disable-PnpDevice -InstanceId "${instanceId}" -Confirm:$false
    Start-Sleep -Seconds 2
    Enable-PnpDevice -InstanceId "${instanceId}" -Confirm:$false
  `;
  execSync(`powershell -Command "${script}"`, { encoding: 'utf-8' });
}
```

### How to Get PnP Instance IDs

```javascript
function getInstanceIds() {
  const script = `
    Get-CimInstance Win32_PnPEntity |
      Where-Object { $_.DeviceID -like "*BTHENUM*110b*" } |
      Select-Object DeviceID, Name, Status |
      ConvertTo-Json
  `;
  return JSON.parse(execSync(`powershell -Command "${script}"`, { encoding: 'utf-8' }));
}
```

### UI/UX Requirements

- **Dark theme** with a modern look (think Windows 11 Settings app aesthetic, or Spotify-like)
- **Sidebar** showing device list with status indicators (green dot = connected, gray = disconnected)
- **Main panel** shows selected device's codec settings
- Only show codec options the device actually supports (check Capability key — if ChannelMode is 0, hide that codec)
- **Live status bar** at the bottom showing: codec, bitrate, latency, connection quality
- Sliders for bitrate/bitpool with real-time value display
- **Apply button** that writes to Next registry key and reconnects
- Toast/notification when a device connects or disconnects
- Should auto-detect new devices when they pair

### Important Notes

- The app needs to **run as Administrator** to write to HKLM registry and to disable/enable PnP devices
- The driver and its service (`AltA2dpSVC.exe`) must already be installed — this app does NOT install drivers
- Values in the Capability key are bitmasks (OR'd together) showing all supported options
- Values in the Next key should be single selections (pick one from the supported bitmask)
- The driver reads from `Next` only on reconnection — that's why we need the disable/enable cycle
- AacBitrate of 0 in the Next key means "use device default" — don't show 0 in the UI, show "Auto" or the Capability's advertised max

### Example Device Data (for testing/mockups)

```json
{
  "name": "Abdullah's AirPods Pro",
  "address": "0000340e224a88bb",
  "instanceId": "BTHENUM\\{0000110B-0000-1000-8000-00805F9B34FB}_VID&0001004C_PID&2027\\9&3A3A251&0&340E224A88BB_C00000000",
  "supports": ["SBC", "AAC"],
  "currentCodec": "AAC",
  "currentBitrate": 256000,
  "connected": true,
  "latencyMs": 150
}
```

Build me this app. Start with the project setup, then the backend (registry reading/writing), then the frontend UI.
