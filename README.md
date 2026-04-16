# Network Security Analysis Lab

## Project Overview
A hands-on simulation of network-level attacks between a **Kali-Linux** attacker and a **Metasploitable 2** target. This project demonstrates the ability to capture, filter, and analyze malicious traffic using **Wireshark**.

## Attack Scenarios Simulated
1. **TCP SYN Port Scanning:** Detected reconnaissance patterns across 15+ ports.
2. **ARP Spoofing (Man-In-The-Middle):** Identified unauthorized Mac to IP mapping and duplicate IP warnings.
3. **Domain Generation Algorithm based C2 Communication:** Scripted a Bash loop to simulate DNS-based exfiltration.

## Tools Used
* **Wireshark** (Packet Analysis)
* **Kali Linux** (Attack Simulation)
* **Metasploitable 2** (Target)
* **Bash** (DGA Scripting)

## Key Findings
* Isolated **2,000+ malicious packets** using custom Wireshark display filters.
* Correlated **Layer 2** and **Layer 3** data to prove identity spoofing.
* Analyzed `SERVFAIL` responses to identify **DNS tunneling** attempts.
