---
title: "Wireshark"
date: "Oct 2026"
summary: "Complete Wireshark notes from beginner to advanced: interface and menus, capture vs display filters, protocol analysis, statistics tools, decryption, TShark, and SOC investigation workflows."
tags:
  - Wireshark
  - Packet Analysis
  - Networking
  - SOC
  - Blue Team
---

# Wireshark: Beginner to Advanced

> **Authorization first.** Only capture traffic on networks you own or have explicit permission to monitor. Packet captures can contain passwords, personal data, and private messages. Treat every `.pcap` as sensitive evidence.

---

## Table of Contents

1. [Fundamentals](#1-fundamentals)
2. [Interface Tour](#2-interface-tour)
3. [Menu Map (What Each Menu Does)](#3-menu-map-what-each-menu-does)
4. [Capturing Traffic](#4-capturing-traffic)
5. [Capture Filters (BPF)](#5-capture-filters-bpf)
6. [Display Filters](#6-display-filters)
7. [Display Filter Library by Protocol](#7-display-filter-library-by-protocol)
8. [Right-Click Menu and Filter Building](#8-right-click-menu-and-filter-building)
9. [Customizing: Columns, Time, Colors, Profiles](#9-customizing-columns-time-colors-profiles)
10. [Following Streams and Navigating](#10-following-streams-and-navigating)
11. [Statistics Menu Explained](#11-statistics-menu-explained)
12. [Expert Information and TCP Analysis](#12-expert-information-and-tcp-analysis)
13. [Protocol Quick Reference](#13-protocol-quick-reference)
14. [TLS and Decryption](#14-tls-and-decryption)
15. [Extracting Files and Data](#15-extracting-files-and-data)
16. [TShark and Command-Line Tools](#16-tshark-and-command-line-tools)
17. [SOC Analyst Playbook: Investigating a PCAP](#17-soc-analyst-playbook-investigating-a-pcap)
18. [SOC Detection Patterns and Filters](#18-soc-detection-patterns-and-filters)
19. [Performance and Big Captures](#19-performance-and-big-captures)
20. [Common Pitfalls and Troubleshooting](#20-common-pitfalls-and-troubleshooting)
21. [Keyboard Shortcuts](#21-keyboard-shortcuts)
22. [Practice Plan](#22-practice-plan)
23. [Cheat Sheet](#23-cheat-sheet)
24. [Key Takeaways](#24-key-takeaways)

---

# Part 1: Beginner

## 1. Fundamentals

**Wireshark** is a graphical packet analyzer. It captures network traffic (or opens saved captures) and decodes it protocol by protocol so you can see exactly what happened on the wire.

Used for:

* Troubleshooting network problems (latency, retransmissions, DNS failures)
* Learning how protocols actually work
* Security investigation (malware traffic, scans, exfiltration, credential theft)
* Verifying that traffic is (or isn't) encrypted

### Key terms

| Term | Meaning |
|---|---|
| Packet / frame | One captured unit of data (Wireshark numbers them in order) |
| Capture | Recording packets from an interface live |
| PCAP / PCAPNG | Capture file formats (pcapng is the modern default, pcap is legacy) |
| Dissector | Wireshark's protocol decoder |
| Capture filter | Decides what gets **recorded** (set before capture, BPF syntax) |
| Display filter | Decides what gets **shown** (applied any time, Wireshark syntax) |
| Stream | All packets of one conversation (TCP, UDP, TLS, HTTP) |
| Promiscuous mode | NIC accepts frames not addressed to it |
| Monitor mode | Wi-Fi mode that captures raw 802.11 frames (needs compatible adapter) |
| SPAN / mirror port | Switch feature that copies traffic to a monitoring port |
| TAP | Hardware device that copies traffic on a link |

### Packet encapsulation (what you're decoding)

```text
Frame (capture metadata)
 └─ Ethernet II   (MAC addresses)
     └─ IP        (IP addresses, TTL, protocol)
         └─ TCP / UDP / ICMP   (ports, flags, sequence numbers)
             └─ Application   (HTTP, DNS, TLS, SMB, ...)
```

The **packet details pane** shows exactly these layers, top (outermost) to bottom (innermost).

### Install and permissions

* **Windows/macOS:** installer; Windows needs **Npcap** (bundled with installer).
* **Linux:** `sudo apt install wireshark`, then allow non-root capture: add your user to the `wireshark` group, log out and back in.
* Capturing runs in a separate privileged helper (**dumpcap**); the GUI itself should not need root.
* Never run unknown pcaps from untrusted sources on a machine you care about. Malicious captures can exploit dissector bugs (rare but real). Use a VM.

---

## 2. Interface Tour

```text
┌──────────────────────────────────────────────────────────────┐
│ Menu bar          File Edit View Go Capture Analyze ...      │
│ Main toolbar      start/stop/restart, open, save, find, zoom │
│ Filter toolbar    [ display filter bar ............ ] Apply  │
├──────────────────────────────────────────────────────────────┤
│ Packet List pane    No. | Time | Source | Dest | Proto | Info│
├──────────────────────────────────────────────────────────────┤
│ Packet Details pane   expandable protocol tree               │
├──────────────────────────────────────────────────────────────┤
│ Packet Bytes pane     hex + ASCII                            │
├──────────────────────────────────────────────────────────────┤
│ Status bar    profile | packets shown/captured | expert info │
└──────────────────────────────────────────────────────────────┘
```

### The three panes

| Pane | What it shows | Tips |
|---|---|---|
| **Packet List** | One row per packet: number, time, source, destination, protocol, length, info | Click column headers to sort. Right-click header to show/hide columns. |
| **Packet Details** | Layer-by-layer decode of the selected packet | Click any field to highlight its bytes. Right-click a field to filter/apply as column. |
| **Packet Bytes** | Raw hex and ASCII | Right-click to switch to bits view; useful for spotting hidden strings or magic bytes. |

### Display filter bar colors

| Color | Meaning |
|---|---|
| **Green** | Valid filter |
| **Red** | Invalid syntax or unknown field |
| **Yellow** | Valid but possibly not what you meant (e.g. deprecated syntax, ambiguous `!=`) |

The filter bar has **autocomplete** and **history** (down-arrow). Use the `+` button at the right to save filters as **filter buttons**.

### Default packet list colors

Wireshark colors rows by protocol/state using **coloring rules** (View > Coloring Rules). In general: TCP and UDP have their own tints, and rows that indicate TCP problems (resets, retransmissions, bad checksums) stand out with darker or red styling. Don't memorize colors; read the **Info** column and **Expert Info** instead. Colors are configurable per profile.

### Status bar

Shows the active **profile**, packet counts (`Packets: 5000 · Displayed: 120`), and the **Expert Information** indicator (colored circle). Click the circle to open Expert Info.

---

## 3. Menu Map (What Each Menu Does)

### File

| Item | Does |
|---|---|
| Open / Open Recent | Load a capture |
| Merge | Combine another capture into the current one |
| Import from Hex Dump | Build a capture from text hex dump |
| Save / Save As | Save the whole capture |
| **Export Specified Packets** | Save only filtered/marked packets as a new pcap |
| **Export Packet Dissections** | Export decoded packet list as text, CSV, JSON, etc. |
| **Export Objects** | Extract files from HTTP, SMB, IMF (email), TFTP, DICOM |
| Export TLS Session Keys | Save TLS secrets (if present) |
| Print | Print packets |

### Edit

| Item | Does |
|---|---|
| Copy | Copy packet info/bytes/fields/filters |
| Find Packet (Ctrl+F) | Search by display filter, hex, string, or regex |
| Find Next / Previous | Repeat search |
| **Mark / Unmark Packet** (Ctrl+M) | Highlight packets of interest |
| **Ignore / Unignore Packet** (Ctrl+D) | Exclude from dissection and stats |
| **Packet Comment** | Add analyst notes to a packet (saved in pcapng) |
| Set/Unset Time Reference (Ctrl+T) | Time column becomes relative to that packet |
| Time Shift | Adjust timestamps |
| Configuration Profiles | Create/switch profiles (columns, filters, colors) |
| Preferences (Ctrl+Shift+P) | All settings (protocols, name resolution, appearance) |

### View

| Item | Does |
|---|---|
| Toolbars / panes toggles | Show/hide interface parts |
| Colorize Packet List | Toggle coloring |
| Coloring Rules | Edit rules |
| Zoom In/Out/Normal | Font size |
| **Time Display Format** | Date/time, seconds since capture start, since previous packet, UTC |
| **Name Resolution** | Resolve MAC, network (DNS), transport names |
| Expand/Collapse Subtrees, Expand All | Packet details tree |
| Resize Columns | Fit columns |
| Reload (Ctrl+R) | Re-read file and reapply |

### Go

| Item | Does |
|---|---|
| Go to Packet (Ctrl+G) | Jump to packet number |
| Go to Corresponding Packet | Jump from a request to its response (and vice versa) |
| Next/Previous Packet | Move in list |
| Next/Previous Packet in Conversation | Move within the same stream |
| First / Last Packet | Jump ends |
| Auto Scroll in Live Capture | Follow new packets |

