# DMXRouter

**Professional DMX512 lighting control router for live entertainment and architectural installations.**

DMXRouter is a high-performance application written in C++ with Qt6 that handles DMX512 data routing, merging, and show management across the major industry protocols. Designed for production environments where reliability and sub-millisecond timing are non-negotiable.

---

## Features at a Glance

- **Multi-protocol routing** — Art-Net 4, sACN (E1.31 2018), with full cross-protocol bridging
- **Universe merge engine** — 8 merge modes including HTP, LTP, Backup, X-Fade, Switch, and Custom per-channel policy
- **Show cue recording & playback** — capture, sequence, and automate DMX shows with crossfade transitions and DMX remote triggering
- **RDM device management** — full E1.20 with device discovery, parameter control, and sensor monitoring
- **RDMNet / LLRP** — E1.33 broker connection and LLRP device discovery
- **Channel-level patching** — per-channel remap, scale (0–200%), min/max limits, CSV import/export
- **Network discovery** — live Art-Net node and sACN source discovery with remote node configuration
- **VLAN management** — Windows Hyper-V virtual switch management for production network segmentation
- **Real-time statistics** — per-interface and per-universe throughput metrics with live event log
- **Profile manager** — save and recall complete configurations
- **~27,000 lines of production C++** — zero compiler warnings with strict flags (`-Wall -Wextra -Wpedantic`)

---

## Table of Contents

- [Architecture](#architecture)
- [Protocol Support](#protocol-support)
- [Merge Engine](#merge-engine)
- [Show Cue System](#show-cue-system)
- [RDM & RDMNet](#rdm--rdmnet)
- [Channel Patching](#channel-patching)
- [Network Discovery](#network-discovery)
- [VLAN Management](#vlan-management)
- [Statistics & Logging](#statistics--logging)
- [Configuration](#configuration)
- [Typical Use Cases](#typical-use-cases)
- [License](#license)

---

## Architecture

DMXRouter runs on a **single-threaded event-loop architecture** driven by Qt's event system. This is a deliberate design decision: crossfade calculations and merge operations complete in microseconds per tick even at high universe counts, while a multi-threaded worker queue would introduce latency through queued connections. The result is consistent sub-millisecond output timing — critical for live shows.

```
┌─────────────────────────────────────────────────────────┐
│                      Qt Event Loop                      │
├──────────────┬──────────────┬─────────────┬────────────┤
│  UDPTransport│  MergeEngine │ShowCueEngine│DiscoveryMgr│
│  (Art-Net /  │  (8 modes,   │(cue record/ │(Art-Net /  │
│   sACN RX/TX)│  512 routes) │ playback)   │ sACN scan) │
├──────────────┴──────────────┴─────────────┴────────────┤
│  RDMManager │ RDMNetManager │ PatchManager│ StatsEngine │
│  (E1.20 RDM)│ (E1.33/LLRP) │ (ch remap)  │ (metrics)  │
├─────────────┴───────────────┴─────────────┴────────────┤
│              Qt6 GUI (MainWindow + Widgets)             │
└─────────────────────────────────────────────────────────┘
```

Key design invariants:

- Per-universe sequence counters (Art-Net and sACN) — compliant with Art-Net 4 §ArtDmx and E1.31 §6.2.6
- Output rate limiter (token bucket at 44 fps / 22.7 ms) — prevents receiver overload
- Socket send/receive buffers at 2 MB — absorbs burst traffic from 40+ simultaneous universes
- Defensive bounds checking throughout — stale indices and corrupted fade states produce log warnings, never visible artifacts on a live rig

---

## Protocol Support

### Art-Net 4
- Receive and transmit ArtDmx on any local network interface
- ArtPoll / ArtPollReply discovery with manual trigger (no background polling traffic on production networks)
- Remote node configuration via ArtAddress and ArtIpProg
- Multi-bind node merging (combines replies from the same IP across ports)
- Correct per-universe sequence numbering (1–255, wrapping, independent per universe)

### sACN — ANSI E1.31 2018
- Full multicast and unicast support
- Per-universe per-source priority (0x64 default, 0xDD per-channel override supported)
- **Universe Synchronization** — E1.31 Extended Sync packets (vector 0x00000001) for glitch-free multi-universe refresh on LED walls and large installations
- Universe Discovery (10-second cycle with pagination)
- Stream termination handling

### Cross-protocol bridging
Any input protocol can be routed to any output protocol. Art-Net → sACN, sACN → Art-Net, or same-protocol universe remapping — all configurable per route.

### Internal Routing (Process Engine Chaining)
The output of one merge engine can be fed as the input to another, enabling cascading merge topologies. A recursion guard (depth 4) prevents infinite loops.

---

## Merge Engine

Each merge engine accepts **up to 4 inputs** and produces one merged output. Up to **512 engines** can run simultaneously.

### Merge Modes

| Mode | Description |
|------|-------------|
| **HTP** | Highest Takes Precedence — maximum value per channel across all inputs |
| **LTP** | Latest Takes Precedence — most recently updated source wins per channel |
| **Backup** | Primary input active; secondary takes over automatically when primary times out |
| **X-Fade** | Crossfade between two sources via a DMX control channel (0 = input 1, 255 = input 2) |
| **Switch** | Select one of up to 4 inputs via DMX control values (8–15 = input 1, 16–23 = input 2…) |
| **Custom** | Per-channel merge policy — each of the 512 channels independently set to Input1/2/3/4, HTP, or LTP |
| **sACN Priority** | Merges sources using E1.31 per-universe priority values |
| **Preset / Snapshot** | Startup buffer that holds the last known state across power cycles |

### Per-engine Features

- **Master / Limit** — scale the entire output (0–100%) and set per-channel hard limits
- **Source IP filter** — accept data only from specific IP addresses
- **Startup buffer** — send a stored snapshot while waiting for live sources to appear
- **Failsafe** — configurable behaviour when all sources time out (hold last / go to black / send preset)
- **Channel patch** — per-channel remap applied after merge, before transmission

---

## Show Cue System

DMXRouter includes a complete show programming and playback engine for automated lighting control.

### Recording
- Capture a snapshot of the live merged DMX output across all active process engines as a numbered cue
- Up to **40 shows**, each with an unlimited cue list
- Each cue stores per-engine DMX state (512 channels × engine count)
- Cues carry individual fade times and user labels

### Playback
- **Go** — advance to the next cue with a smooth crossfade
- **Jump** — go to any cue by index instantly
- **Back / Forward** — pre-select the next cue without triggering
- **Pause / Resume** — pause mid-crossfade; the fade resumes from exactly where it was stopped
- **Stop** — halt playback immediately and inject a blackout
- **Hold timer** — configurable auto-advance delay before the next cue fires

### Crossfade Engine
- 40 Hz update rate (25 ms tick) for smooth transitions
- Linear interpolation across all 512 channels of every active engine
- Flash-free cue jumps after a Stop state (skips the crossfade to prevent ghost-frame artifacts)

### DMX Remote Control
- Any DMX channel on any universe can trigger record, play, stop, and cue selection from a lighting desk
- Arm / disarm prevents accidental triggers on startup
- Gap guard prevents cue flooding from noisy DMX faders

---

## RDM & RDMNet

### RDM — ANSI E1.20
- Discover devices on any Art-Net universe (ArtRdm packets)
- Identify, set DMX start address, device label, and personality
- Read 19+ PIDs: device info, manufacturer, model, personality list, DMX address, identify state, sensor definitions and values, lamp state, lamp on mode, product detail, supported parameters, and more
- 3-second transaction timeout with automatic retry
- Full device cache with parameter persistence
- Interactive device tree in the **🎛 RDM** tab

### RDMNet — ANSI E1.33 / LLRP
- **LLRP discovery** — multicast probe on 239.255.250.133 and 239.255.250.134 with known-UID suppression
- **RDM over LLRP** — send RDM commands to LLRP targets without an Art-Net path
- **Broker connection** — TCP with full Client Connect handshake, 15-second heartbeat, Client Fetch List, RPT Request/Notification/Status, and broker redirect (IPv4 and IPv6)
- Self-filtering by CID prevents echo responses
- Dedicated **🌐 RDMNet** tab with LLRP target list, broker controls, and client roster

---

## Channel Patching

Full channel-level remapping applied after merge and before output.

- **512-channel remap** — any input channel to any output channel
- **Scale** — multiply each channel value from 0% to 200%
- **Min / Max clamp** — hard floor and ceiling per channel
- **Bulk operations** — identity reset, channel offset, range map, pair swap, reverse, fan-out, dimmer curve
- **Presets** — save and recall patch configurations
- **CSV import / export** — compatible with standard patch sheets
- **Mini-map** — 32×16 visual overview of the complete 512-channel patch

---

## Network Discovery

The **🔍 Discovery** tab shows all Art-Net nodes and sACN sources visible on the network in real time.

**Art-Net nodes:** short name, long name, firmware version, IP, port count, active universes. Remote configuration via ArtAddress and ArtIpProg directly from the UI. Nodes removed 60 seconds after last reply.

**sACN sources:** source name, CID, IP, universe list. Sources removed 15 seconds after last packet.

ArtPoll is **manually triggered** via toolbar button to avoid continuous background traffic on production networks.

---

## VLAN Management

On Windows 10/11 Pro or Enterprise with Hyper-V, DMXRouter can create and manage virtual network adapters tagged to specific VLANs — equivalent to the Group/Trunk system found in professional lighting network hardware.

- Create / destroy Hyper-V Virtual Switch via asynchronous PowerShell
- Add and remove VLANs with configurable IDs and names
- Colour-coded VLAN table
- Adapter filtering hides system adapters (Default Switch, management NICs)
- Network diagnostics panel
- On non-Windows platforms, a clear advisory guides using native OS tools

> Requires: Windows administrator privileges and Hyper-V feature enabled.

---

## Statistics & Logging

The **📈 Stats & Log** tab provides live operational visibility.

**Metrics dashboard** — 8 live cards: Packets In/s, Packets Out/s, Total In, Total Out, Active Universes, Error Count, Sequence Errors, Uptime. Colour-coded green / red / grey by state.

**Throughput chart** — rolling 2-minute history (120 snapshots), rendered with QPainter. Cyan for inbound, green for outbound, semi-transparent fill, auto-scaling Y axis.

**Per-interface breakdown** — packet counts, Art-Net / sACN protocol split, error totals.

**Per-universe breakdown** — packet rates, merge operation counts, sequence errors, last-seen timestamp.

**Event log** — ring buffer of 10,000 entries, thread-safe. Captures all `qDebug` / `qInfo` / `qWarning` / `qCritical` output. Automatic category tagging (ArtNet, sACN, Transport, Merge, Discovery, Network, System). Filterable by level and category. Auto-scroll toggle, Clear button, monospace font.

---

## Configuration

All settings are saved to a single JSON file via **File → Save Config** (Ctrl+S) and restored with **File → Load Config** (Ctrl+O). The file includes: routing table, merge engine configurations, channel patches, show cues, VLAN settings, and discovery preferences.

**Profile Manager** allows saving named snapshots of the complete configuration for quick recall during productions.

Example configuration excerpt:

```json
{
  "merges": [
    {
      "name": "FOH Merge",
      "output": { "protocol": "sacn", "universe": 1 },
      "mode": "htp",
      "inputs": [
        {
          "source": { "protocol": "artnet", "net": 0, "subnet": 0, "universe": 0 },
          "interface": "eth0:10.0.0.1",
          "priority": 100
        }
      ],
      "master": 100,
      "failsafe": "hold"
    }
  ],
  "routes": [
    {
      "name": "Main to sACN",
      "input":  { "protocol": "sacn",   "universe": 1 },
      "output": { "protocol": "artnet", "net": 0, "subnet": 0, "universe": 0 },
      "output_interface": "eth0:10.0.0.1",
      "sacn_sync_address": 0,
      "enabled": true
    }
  ]
}
```

---

## Typical Use Cases

**Protocol bridge** — receive Art-Net from a console on one NIC, output sACN to fixtures on another, or the reverse. Each interface is independently addressed.

**Console merge** — two FOH consoles sending to the same universe, merged via HTP so the highest value from either wins at all times.

**Priority backup failover** — main console at sACN priority 200, backup at 100. If the main drops, the backup takes over automatically within the source timeout window.

**X-Fade / media server handoff** — crossfade between a lighting desk and a media server output using a single DMX fader as the blend control.

**Show automation** — record cue snapshots during rehearsal and play them back during the performance with configurable crossfade times, triggered manually, on a timer, or from a DMX control channel on the desk.

**Fixture repatching** — remap RGB → BGR for fixtures wired in a non-standard order, or offset a dimmer rack that starts at an unusual DMX address, without touching the console patch.

**Network segmentation** — receive on a production VLAN, output on a separate lighting VLAN, using DMXRouter as the bridge between segments.

**Large LED installation** — use sACN Universe Synchronization to ensure all universes are released simultaneously at receiver endpoints, eliminating visible tearing across a multi-universe LED wall.

---

## License

Copyright (c) 2026. All rights reserved.

This software is proprietary. See the [LICENSE](LICENSE) file for full terms.

This application uses **Qt 6**, licensed under the LGPL v3. Qt is dynamically linked and unmodified. See the [NOTICE](NOTICE) file for third-party attributions and your rights under the LGPL.

---

*DMXRouter v1.0 — Built for the stage.*
