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

### Capture

| Item | Does |
|---|---|
| Options (Ctrl+K) | Interface, capture filter, promiscuous, buffer, ring buffer, stop conditions |
| Start / Stop / Restart (Ctrl+E / Ctrl+R) | Control capture |
| Capture Filters | Manage saved capture filters |
| Refresh Interfaces | Re-detect NICs |

### Analyze

| Item | Does |
|---|---|
| Display Filters / Macros / Expression | Manage saved filters and macros |
| **Apply as Column** (Ctrl+Shift+I) | Add selected field as a column |
| **Apply as Filter / Prepare as Filter** | Build filters from selected field |
| **Conversation Filter** | Filter on the conversation of the selected packet |
| Enabled Protocols | Turn dissectors on/off |
| **Decode As** | Force a protocol on a non-standard port |
| **Follow** | Follow TCP/UDP/TLS/HTTP stream |
| Show Packet Bytes | Raw view in new window |
| **Expert Information** | Warnings and errors summary |

### Statistics

See [section 11](#11-statistics-menu-explained).

### Telephony / Wireless / Tools

| Menu | Highlights |
|---|---|
| Telephony | VoIP Calls, SIP Flows, RTP Streams (can play captured audio) |
| Wireless | Bluetooth and WLAN traffic summaries |
| Tools | Firewall ACL Rules, Credentials (newer versions list cleartext creds found), Lua |

---

# Part 2: Intermediate

## 4. Capturing Traffic

### Choosing where to capture

| Method | Sees | Notes |
|---|---|---|
| Your own NIC | Your traffic (and broadcast/multicast) | On a switched network you won't see others' unicast traffic |
| SPAN / mirror port | Copy of chosen port/VLAN traffic | Typical SOC/NOC method |
| Network TAP | Full copy of a link | Most reliable, no packet loss on busy links |
| Server-side `tcpdump` / `dumpcap` | Traffic at that host | Then copy the pcap to analyze |
| Wi-Fi monitor mode | Raw 802.11 frames | Needs compatible adapter and driver |

### Capture Options dialog (Ctrl+K)

| Setting | What it does |
|---|---|
| Interface | Which NIC to capture on (the sparkline shows live activity) |
| **Promiscuous mode** | Capture frames not addressed to you (limited on switched/Wi-Fi networks) |
| Monitor mode | Wi-Fi raw capture (Linux/macOS mostly) |
| **Capture filter** | BPF filter applied during capture |
| Snapshot length (snaplen) | Max bytes saved per packet (default is full; lowering saves space but truncates payloads) |
| Buffer size | Kernel buffer (increase if dropping packets) |
| **Output: file / multiple files** | Save to disk, optionally ring buffer |
| Ring buffer | Keep last N files, rotate automatically (great for long monitoring) |
| Stop capture after... | N packets, size, or duration |
| Name resolution | MAC / network / transport lookups (network resolution generates DNS traffic) |
| Update list in real time / auto scroll | Live display behavior |

### Good capture habits

* Capture to a file (pcapng) so nothing is lost if the app crashes.
* Note the **time zone**, the **capture point**, and **who/what the capture represents**.
* Keep captures **as small as the question allows** (filters, short windows).
* Hash the pcap for chain of custody: `sha256sum capture.pcapng`.
* Turn off network name resolution when investigating (avoid leaking your lookups and keep IPs raw).

---

## 5. Capture Filters (BPF)

Capture filters use **Berkeley Packet Filter (BPF)** syntax, the same as `tcpdump`. They are set **before** capture and decide what is recorded. They are efficient but **less flexible** than display filters, and you **cannot change them mid-capture**.

### Syntax building blocks

| Piece | Examples |
|---|---|
| Type | `host`, `net`, `port`, `portrange` |
| Direction | `src`, `dst`, `src or dst` (default both) |
| Protocol | `tcp`, `udp`, `icmp`, `arp`, `ip`, `ip6`, `vlan` |
| Operators | `and` (`&&`), `or` (`\|\|`), `not` (`!`), parentheses |

### Examples

```text
host 192.168.1.10                   traffic to or from this host
src host 192.168.1.10               only from this host
dst host 8.8.8.8                    only to this host
net 192.168.1.0/24                  whole subnet
port 80                             port 80 (TCP or UDP)
tcp port 443                        TCP 443 only
udp port 53                         DNS
portrange 1-1024                    well-known ports
not port 22                         exclude SSH (useful when capturing over SSH!)
host 10.0.0.5 and tcp port 80       combine
(tcp port 80 or tcp port 443) and host 10.0.0.5
icmp                                ICMP only
arp                                 ARP only
not arp and not broadcast           skip noise
ether host aa:bb:cc:dd:ee:ff        by MAC
tcp[tcpflags] & tcp-syn != 0        packets with SYN flag set
tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn   SYN without ACK (scan/connect attempts)
less 128                            packets under 128 bytes
greater 1000                        packets over 1000 bytes
vlan 100                            specific VLAN
```

> **Rule of thumb:** Use a capture filter to avoid recording junk (and your own SSH/RDP session). Use display filters for everything else, so you don't permanently discard data you may need later.

---

## 6. Display Filters

Display filters hide non-matching packets in the list. They **do not delete** anything, and you can change them at any time. Syntax is different from BPF.

### Operators

| Operator | Alternate | Meaning | Example |
|---|---|---|---|
| `==` | `eq` | Equal | `ip.addr == 10.0.0.5` |
| `!=` | `ne` | Not equal (see gotcha below) | `tcp.port != 22` |
| `>` | `gt` | Greater than | `frame.len > 1000` |
| `<` | `lt` | Less than | `ip.ttl < 10` |
| `>=` | `ge` | Greater or equal | `tcp.window_size >= 65535` |
| `<=` | `le` | Less or equal | `tcp.len <= 0` |
| `contains` | | Field contains string/bytes (case-sensitive) | `http.host contains "evil"` |
| `matches` | `~` | Regex match | `http.request.uri matches "(?i)union.+select"` |
| `in` | | Value in a set | `tcp.port in {80 443 8080}` |
| `&&` | `and` | Both true | `ip.src == 10.0.0.5 && tcp` |
| `\|\|` | `or` | Either true | `dns \|\| http` |
| `!` | `not` | Negate | `!arp` |
| `[a:b]` | | Slice (offset:length) | `eth.src[0:3] == 00:0c:29` |

### Basics: filter by protocol, address, port

```text
http                      all HTTP
dns                       all DNS
tcp                       all TCP
udp                       all UDP
icmp                      all ICMP
arp                       all ARP
tls                       TLS (older versions used "ssl")

ip.addr == 192.168.1.10           either source or destination
ip.src == 192.168.1.10            source only
ip.dst == 192.168.1.10            destination only
ip.addr == 192.168.1.0/24         whole subnet
eth.addr == aa:bb:cc:dd:ee:ff     MAC
tcp.port == 443                   either source or destination port
tcp.srcport == 443                source port
tcp.dstport == 443                destination port
udp.port == 53
frame.len > 1000                  large frames
frame contains "password"         string anywhere in the frame
tcp contains "admin"              string in TCP layer
```

### Combining

```text
ip.addr == 10.0.0.5 && tcp.port == 443
(http || dns) && ip.src == 10.0.0.5
tcp.port in {80 443 8080 8443}
ip.addr in {10.0.0.5 10.0.0.9}
!(ip.addr == 10.0.0.0/8)                   exclude RFC1918 range 10.x
ip.src == 10.0.0.0/8 && !(ip.dst == 10.0.0.0/8)   internal -> external only
```

### Time-based

```text
frame.time >= "2026-10-26 10:00:00" && frame.time <= "2026-10-26 10:30:00"
frame.time_delta > 1                       gap > 1s since previous captured packet
frame.time_relative > 60                   after 60s from capture start
```

### Big gotcha: `!=` with `ip.addr`

`ip.addr != 1.2.3.4` does **not** do what you expect on many versions, because `ip.addr` matches *either* source or destination. Packets where one side is 1.2.3.4 and the other isn't can still match.

```text
WRONG:   ip.addr != 1.2.3.4
RIGHT:   !(ip.addr == 1.2.3.4)
```

Same for `tcp.port`, `udp.port`, `eth.addr`. Use `ip.src != x && ip.dst != x` or the negated parentheses form.

### Other tips

* Strings are **case-sensitive** with `contains`. Use `matches "(?i)text"` for case-insensitive.
* Quote strings: `http.host == "example.com"`.
* Hex bytes use colons: `eth.src == 00:11:22:33:44:55`.
* Find field names: click a field in packet details, status bar shows the filter name. Or Analyze > Display Filter Expression. Or `tshark -G fields | grep dns`.
* Click a field > right-click > **Apply as Filter** to avoid guessing syntax.

---

## 7. Display Filter Library by Protocol

### IP / ICMP / ARP

```text
ip.ttl < 10                         low TTL (far away or traceroute)
ip.ttl == 128                       likely Windows origin
ip.flags.mf == 1 || ip.frag_offset > 0    fragmented packets
ip.version == 6                     IPv6
icmp.type == 8                      echo request (ping)
icmp.type == 0                      echo reply
icmp.type == 3                      destination unreachable
icmp.type == 3 && icmp.code == 3    port unreachable (UDP scan evidence)
icmp.type == 11                     TTL exceeded (traceroute)
icmp && data.len > 64               unusually large ICMP payload (tunneling suspect)
arp.opcode == 1                     ARP request
arp.opcode == 2                     ARP reply
arp.duplicate-address-detected      same IP claimed by multiple MACs
arp.src.proto_ipv4 == 10.0.0.1      ARP about a specific IP
```

### TCP

```text
tcp.flags.syn == 1 && tcp.flags.ack == 0     connection attempts (SYN only)
tcp.flags.syn == 1 && tcp.flags.ack == 1     SYN/ACK (port open responses)
tcp.flags.reset == 1                          resets
tcp.flags.fin == 1                            closes
tcp.flags == 0x000                            NULL flags
tcp.flags == 0x029                            Xmas (FIN+PSH+URG)
tcp.stream eq 5                               a specific TCP conversation
tcp.len > 0                                   packets carrying data
tcp.window_size == 0                          zero window (receiver overwhelmed)
tcp.analysis.flags                            any TCP problem Wireshark detected
tcp.analysis.retransmission                   retransmissions
tcp.analysis.fast_retransmission
tcp.analysis.duplicate_ack
tcp.analysis.out_of_order
tcp.analysis.lost_segment
tcp.analysis.zero_window
tcp.analysis.window_full
tcp.analysis.reused_ports                     port reuse
tcp.time_delta > 1                            slow responses within a stream
tcp.port == 4444                              classic reverse-shell/Metasploit port
```

### UDP / DNS

```text
udp.length > 512
dns                                      all DNS
dns.flags.response == 0                  queries only
dns.flags.response == 1                  responses only
dns.qry.name == "example.com"
dns.qry.name contains "evil"
dns.qry.name.len > 50                    long names (tunneling / DGA suspect)
dns.qry.type == 1                        A
dns.qry.type == 28                       AAAA
dns.qry.type == 16                       TXT (often abused for C2/exfil)
dns.qry.type == 15                       MX
dns.qry.type == 255                      ANY
dns.flags.rcode == 3                     NXDOMAIN (many = DGA or typos)
dns.flags.rcode != 0                     any DNS error
dns.resp.name contains "evil"
dns.a == 1.2.3.4                         responses with that A record
dns.count.answers > 10
mdns || llmnr || nbns                    local name resolution traffic
```

### HTTP

```text
http.request                                         all requests
http.response                                        all responses
http.request.method == "GET"
http.request.method == "POST"
http.host == "example.com"
http.host contains "malicious"
http.request.uri contains "login"
http.request.full_uri contains "admin"
http.user_agent contains "curl"
http.user_agent contains "python"                    scripted clients
http.response.code == 200
http.response.code == 404
http.response.code >= 400
http.response.code == 401                            auth required/failed
http.response.code == 403
http.referer contains "evil"
http.cookie
http.authorization                                   Authorization header (Basic auth is base64!)
http.content_type contains "application/x-msdownload"   Windows executables
http.content_type contains "octet-stream"
http.file_data contains "This program cannot be run in DOS mode"   PE file in body
http.request.uri matches "(?i)(union.+select|<script|\.\./)"   SQLi/XSS/traversal hints
http.content_length > 1000000
```

### TLS / HTTPS

```text
tls                                                  all TLS
tls.handshake.type == 1                              Client Hello
tls.handshake.type == 2                              Server Hello
tls.handshake.type == 11                             Certificate
tls.handshake.extensions_server_name                 SNI present
tls.handshake.extensions_server_name contains "evil"
tls.handshake.version == 0x0303                      TLS 1.2 in handshake
tls.record.version == 0x0301                         old TLS 1.0 records
tls.alert_message                                    TLS alerts (errors)
x509ce.dNSName contains "example"                    names in certificate
tls.handshake.ja3                                    JA3 fingerprint (available in recent versions)
```

> In TLS 1.3 the certificate is encrypted, so you won't see it in the handshake. SNI in the Client Hello is usually still visible (unless ECH is used).

### SMB / Windows / AD

```text
smb || smb2                                  SMB traffic
smb2.cmd == 5                                Create (file open)
smb2.cmd == 3                                Tree Connect
smb2.tree contains "IPC$"                    IPC share
smb2.tree contains "ADMIN$"                  admin share (lateral movement indicator)
smb2.filename contains ".exe"
smb2.nt_status == 0xc000006d                 logon failure
smb2.nt_status != 0                          any SMB error
dcerpc                                       RPC calls
svcctl                                       Service Control (remote service creation)
ntlmssp                                      NTLM authentication
ntlmssp.auth.username
kerberos                                     Kerberos traffic
kerberos.CNameString                         username in Kerberos
kerberos.error_code                          Kerberos errors
ldap                                         LDAP queries
dcerpc.cn_bind_to_uuid                       RPC interface bound (research UUID meaning)
```

### Email / FTP / Telnet / SSH / others

```text
ftp                                          FTP
ftp.request.command == "USER"
ftp.request.command == "PASS"                cleartext FTP password
ftp.response.code == 530                     login failed
ftp.response.code == 230                     login success
telnet                                       cleartext remote shell
smtp                                         email sending
smtp.req.command == "AUTH"
imf                                          email message content (Internet Message Format)
pop                                          POP3
imap                                         IMAP
ssh                                          SSH (encrypted, but banners visible)
ssh.protocol                                 SSH version banner
rdp                                          Remote Desktop
dhcp                                         DHCP (older versions used "bootp")
dhcp.option.dhcp == 1                        DHCP Discover
dhcp.option.dhcp == 2                        DHCP Offer (check for multiple servers)
dhcp.option.hostname                         device hostnames
snmp                                         SNMP
ntp                                          NTP
tftp                                         TFTP
syslog                                       Syslog
```

---

## 8. Right-Click Menu and Filter Building

You rarely need to type filters from scratch. Click a field and let Wireshark build the filter.

### Packet details: right-click a field

| Option | What it does |
|---|---|
| **Apply as Filter > Selected** | Filters instantly on that field's value |
| **Apply as Filter > Not Selected** | Filters out that value |
| **...and Selected / ...or Selected** | Adds to the existing filter with AND/OR |
| **...and not Selected / ...or not Selected** | Adds a negation |
| **Prepare as Filter** | Same options but **only types it in the bar** so you can edit before applying |
| **Apply as Column** | Adds that field as a packet list column |
| **Conversation Filter** | Filter for the conversation (Ethernet, IP, TCP, UDP...) |
| **Follow** | Follow TCP/UDP/TLS/HTTP stream |
| Copy > Value / Field Name / As Filter | Copy pieces |
| Expand / Collapse Subtrees | Navigate the tree |
| Protocol Preferences | Per-protocol settings |
| Decode As | Force different decoding for this port |

### Packet list: right-click a packet

| Option | What it does |
|---|---|
| Mark / Ignore Packet | Highlight or exclude |
| Set Time Reference | Reset relative time |
| Packet Comment | Add a note |
| Colorize Conversation | Temporarily color a conversation |
| Show Packet in New Window | Detached view |
| Export Packet Bytes | Save raw bytes |

### Filter workflow tip

Click a field > **Prepare as Filter** > edit > Enter. This teaches you field names while you work.

---
