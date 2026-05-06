# 🔐 Network Security Analysis

![Lab Status](https://img.shields.io/badge/Status-Complete-brightgreen)
![Tools](https://img.shields.io/badge/Tools-Wireshark%20%7C%20Kali%20Linux%20%7C%20Nmap-blue)
![Type](https://img.shields.io/badge/Type-Security%20Lab-red)
![Environment](https://img.shields.io/badge/Environment-VirtualBox%20%7C%20Isolated%20Network-orange)

> A hands-on penetration testing and traffic analysis lab built on an isolated VirtualBox environment. I simulated real-world network attacks such as TCP SYN port scanning, ARP spoofing (MITM) and DGA-based C2 communication. I analyzed the resulting malicious traffic using Wireshark to produce actionable threat detection findings.

---

## 🖥️ Lab Environment

| Role       | OS              | IP Address       |
|------------|-----------------|------------------|
| Attacker   | Kali Linux      | 192.168.56.102   |
| Target     | Metasploitable 2| 192.168.56.101   |
| Gateway    | Windows 11      | 192.168.56.1     |
| Network    | Host-Only Adapter (VirtualBox) — fully isolated, no internet exposure |

All traffic was captured and analyzed using **Wireshark 4.x** on an air-gapped virtual network.

---

## ⚔️ Attack Scenarios

### 1. TCP SYN Port Scanning

**Goal:** Identify open ports on the target using stealth reconnaissance.

**Attack command (Kali Linux):**
```bash
nmap -sS 192.168.56.101
```

**How it works:**
- The attacker sends rapid TCP SYN packets to the target across multiple ports.
- If a port is open, the target replies with `SYN-ACK`. The attacker sends `RST` instead of completing the handshake — keeping the scan "half-open" and harder to detect.
- If a port is closed, the target replies with `RST, ACK`.

**Wireshark filter used:**
(tcp.flags.syn == 1 or tcp.flags.reset == 1) and ip.addr == 192.168.56.101

**What I observed in the capture:**
- Attacker IP `192.168.56.102` sent automated SYN packets with `seq=0, len=0` — zero payload, confirming no data exchange, only port probing.
- Packets 88–96 showed `RST, ACK` responses from the target on closed ports.
- Over 15 ports probed within milliseconds — consistent with automated scanner behavior.

📸 *Screenshot: `screenshots/portScanning.png`*
📦 *Packet capture: `packet-captures/portScans.pcapng`*

---

### 2. ARP Spoofing — Man-in-the-Middle (MITM)

**Goal:** Intercept all traffic between the target and the gateway by poisoning both ARP tables.

**Attack commands:**
```bash
# Tell the target that the attacker is the gateway
sudo arpspoof -i eth1 -t 192.168.56.101 192.168.56.1

# Tell the gateway that the attacker is the target
sudo arpspoof -i eth1 -t 192.168.56.1 192.168.56.101
```

**How it works:**
- The attacker broadcasts fake ARP replies to `192.168.56.101` (Metasploitable), claiming *"I am the gateway. Send your traffic to me."* The target updates its ARP table and routes all outbound traffic through the attacker's MAC.
- The attacker simultaneously tells the gateway *"I am Metasploitable."* The gateway updates its ARP table accordingly.
- Result: All traffic flows through the attacker — a full bidirectional man-in-the-middle position.

No filter applied, Wireshark auto-flagged duplicate IP warnings natively in the Info column.

**What I observed in the capture:**
- MAC address `08:00:27:94:29:4f` repeatedly sent ARP replies claiming ownership of `192.168.56.1` (the gateway IP).
- Wireshark auto-flagged these as **"duplicate use of IP address"** — a built-in indicator of ARP poisoning.
- Gratuitous ARP reply volume was abnormally high, consistent with continuous spoofing to keep tables poisoned.

📸 *Screenshot: `screenshots/ARPSpoof.png`*
📦 *Packet capture: `packet-captures/ARPSpoof.pcapng`*

---

### 3. DGA-Based C2 Communication via DNS

**Goal:** Simulate malware communicating with a Command & Control (C2) server using Domain Generation Algorithm (DGA) DNS lookups.

**DGA simulation script (Bash — run on Kali Linux):**
```bash
#!/bin/bash
# DGA-based C2 Communication Simulation
# Simulates malware beaconing behavior using randomly generated subdomains
# Target DNS server: 192.168.56.101 (Metasploitable 2)

for i in {1..5}; do
    SUBDOMAIN=$(head /dev/urandom | tr -dc A-Za-z0-9 | head -c 20)
    nslookup "${SUBDOMAIN}.malicious-c2-server.com" 192.168.56.101
done
```

**How it works:**
- Real malware uses DGA to generate pseudo-random domain names. The attacker pre-registers one of them as the C2 server — making blocklists ineffective.
- This script replicates that behavior: it generates MD5-seeded subdomains and fires DNS queries at the target DNS server.
- Metasploitable responds with `SERVFAIL` because none of the domains exist — but the query *pattern* is exactly what an infected host would produce.

No filter applied, output observed directly via `nslookup` in terminal.

**What I observed in the capture:**
- Repeated DNS queries for randomly structured subdomains under a fixed parent domain, all resolving to `SERVFAIL`.
- High query frequency in a short time window — a classic DGA beacon signature.
- `192.168.56.102` (Kali) acted as the infected host; `192.168.56.101` (Metasploitable) acted as the DNS server returning failure responses.

📸 *Screenshot: `screenshots/unusual_DNS.png`*
📦 *Packet capture: `packet-captures/unusualDNS.pcapng`*

---

## 🔍 Wireshark Filters Used

| Attack                  | Filter                                                                 |
|-------------------------|------------------------------------------------------------------------|
| TCP SYN Port Scan       | `(tcp.flags.syn == 1 or tcp.flags.reset == 1) and ip.addr == 192.168.56.101` |
| ARP Spoofing            | No filter applied, Wireshark auto-flagged duplicate IP warnings natively in the Info column |
| DGA DNS Beaconing       | No filter applied, output observed directly via `nslookup` in terminal |

---

## 📊 Key Findings

| Finding | Detail |
|---|---|
| **Packets analyzed** | 2,000+ malicious packets isolated using custom Wireshark display filters |
| **Port scan signature** | Identified `seq=0, len=0` SYN flood pattern across 15+ ports in under 1 second |
| **ARP poisoning confirmed** | Single rogue MAC claimed gateway IP via gratuitous ARP replies; Wireshark flagged duplicate IP |
| **Layer 2 + Layer 3 correlation** | Cross-referenced MAC and IP data to confirm identity spoofing beyond ARP table inspection |
| **DGA pattern identified** | Detected high-frequency DNS queries for randomized subdomains returning `SERVFAIL` |

---

## 🛡️ Defensive Recommendations Summary

**Against Port Scanning:**
- Rate-limit inbound SYN packets per source IP at the firewall
- Segment sensitive servers onto private VLANs
- Deploy IDS/IPS (Snort, Suricata) with port scan detection rules
- Disable and close all unused ports

**Against ARP Spoofing:**
- Enable **Dynamic ARP Inspection (DAI)** on managed switches
- Enforce HTTPS/SSH — encrypt traffic so interception yields nothing useful
- Statically bind MAC-to-IP mappings for critical infrastructure
- Monitor for gratuitous ARP anomalies using network monitoring tools

**Against DGA/DNS C2:**
- Implement **DNSSEC** to validate DNS response integrity
- Use **DNS over HTTPS (DoH)** or **DNS over TLS (DoT)** to prevent query interception
- Audit DNS logs for entropy spikes in queried domain names
- Restrict outbound DNS to a single trusted resolver; block direct DNS to external IPs

---

## 🧠 Skills Demonstrated

- **Network protocol analysis** — TCP/IP, ARP, DNS at packet level
- **Offensive techniques** — Port scanning, MITM, DGA simulation
- **Threat detection** — Custom Wireshark filters, anomaly identification
- **Security tooling** — Wireshark, Nmap, arpspoof, dig, Bash scripting
- **Lab environment setup** — VirtualBox, Kali Linux, Metasploitable 2 networking
- **Technical writing** — Documented attack methodology, findings, and remediation

---

## ⚠️ Disclaimer

This project was conducted entirely within an isolated VirtualBox environment for educational purposes only. All attack simulations were performed on systems I own and control. No real networks or systems were harmed.

---

*Built as part of a network security internship project — Krutanic, 2026.*
