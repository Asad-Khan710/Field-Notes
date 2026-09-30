---

title: "Nmap"
date: "Sep 2026"
summary: "Notes on Nmap scanning, port discovery, service detection, and basic reconnaissance."
tags:

- Nmap
- Networking
- Reconnaissance

---

# Nmap Basics

## What is Nmap?

Nmap (Network Mapper) is a network scanning tool used to discover hosts, identify open ports, detect services, and gather information about systems on a network.

It is commonly used during the reconnaissance and enumeration stages of security testing.

## Basic Scan

A basic Nmap scan can be performed with:

```bash
nmap 127.0.0.1
```

This scans the target for commonly used ports.

Example:

```text
PORT     STATE  SERVICE
22/tcp   open   ssh
80/tcp   open   http
```

## Port States

Nmap commonly reports ports using these states:

* Open - A service is listening and accepting connections.
* Closed - The port is reachable, but no service is currently listening.
* Filtered - Nmap cannot determine whether the port is open because filtering is blocking the scan.

## Service and Version Detection

The `-sV` option attempts to identify the service and version running on an open port.

```bash
nmap -sV 127.0.0.1
```

This can provide information such as:

```text
22/tcp   open   ssh     OpenSSH
80/tcp   open   http    Apache
```

Service detection is useful during enumeration because knowing what software is running can help identify potential security issues.

## OS Detection

Nmap can attempt to identify the operating system of a target using:

```bash
nmap -O 127.0.0.1
```

OS detection works by analyzing characteristics of the target's network responses.

Results are not always exact, especially when firewalls or other network controls interfere with the scan.

## Scanning Specific Ports

A specific port can be scanned with the `-p` option.

```bash
nmap -p 22 127.0.0.1
```

Multiple ports can also be specified:

```bash
nmap -p 22,80,443 127.0.0.1
```

## Port Ranges

A range of ports can be scanned:

```bash
nmap -p 1-1000 127.0.0.1
```

This scans ports 1 through 1000.

## Scanning Common Ports

Nmap can scan the most commonly used ports with:

```bash
nmap --top-ports 20 127.0.0.1
```

The number controls how many of the most common ports are scanned.

## Useful Nmap Options

Some useful options include:

```text
-sV          Service and version detection
-O           Operating system detection
-p           Specify ports
--top-ports  Scan common ports
-Pn          Skip host discovery
-v           Increase scan output
```

Example:

```bash
nmap -sV -p 22,80,443 127.0.0.1
```

This scans ports 22, 80, and 443 while attempting to identify the services and versions.

## Common Ports

Some commonly encountered ports include:

* 21 - FTP
* 22 - SSH
* 23 - Telnet
* 25 - SMTP
* 53 - DNS
* 80 - HTTP
* 110 - POP3
* 143 - IMAP
* 443 - HTTPS
* 445 - SMB
* 3389 - RDP

Knowing common ports makes Nmap results easier to understand during enumeration.

## Reconnaissance and Enumeration

Nmap is useful during the reconnaissance and enumeration stages of a security assessment.

A basic workflow can be:

```text
1. Identify the target
2. Discover available ports
3. Identify services
4. Identify service versions
5. Gather additional information
6. Investigate potential vulnerabilities
```

Nmap itself is primarily a discovery and enumeration tool. Finding an open port does not automatically mean that the service is vulnerable.

## Basic Scanning Workflow

A simple progression for learning Nmap is:

```bash
nmap 127.0.0.1
```

Then:

```bash
nmap -sV 127.0.0.1
```

Then:

```bash
nmap -O 127.0.0.1
```

Finally, specific ports can be investigated:

```bash
nmap -sV -p 22,80,443 127.0.0.1
```

Each scan provides additional information about the target.

## Key Takeaways

* Nmap is used for network discovery and enumeration.
* Open ports can reveal available network services.
* `-sV` helps identify service versions.
* `-O` attempts to identify the operating system.
* `-p` allows specific ports or port ranges to be scanned.
* `--top-ports` scans commonly used ports.
* Scan results should be interpreted carefully because firewalls and filtering can affect results.
* Nmap should only be used against systems that you are authorized to scan.

## Security Note

Only scan systems and networks that you own or have explicit permission to test. Unauthorized scanning can violate organizational policies or applicable laws.
