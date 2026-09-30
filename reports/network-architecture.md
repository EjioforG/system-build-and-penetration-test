**ACME SOLUTION INDUSTRY**

**NETWORK ARCHITECTURE DOCUMENT**

**Vulnerable Network Lab — Internal Infrastructure Design**

| Document Title | Network Architecture — Vulnerable Lab Infrastructure |

| --- | --- |

| Organisation | ACME Solution Industry |

| Project | Project 1 — Build & Attack a Vulnerable Network Lab |

| Authors | Ejiofor Godswill \| Demilada |

| Assessment Period | 1 July 2026 – 8 July 2026 |

| Version | 1.0 — Final |

| Classification | CONFIDENTIAL — FOR INTERNAL USE ONLY |

ACME Solution Industry — Vulnerable Network Lab — July 2026

## 1. DOCUMENT PURPOSE

This document describes the network architecture and infrastructure design of the deliberately vulnerable network environment constructed by ACME Solution Industry as part of Project 1 — Build & Attack a Vulnerable Network Lab. The environment was designed to simulate a realistic enterprise network containing intentional security flaws, enabling a comprehensive penetration test to be conducted against it by a peer security team.

The architecture implements a three-tier network design consisting of a WAN-facing external segment, a De-Militarised Zone (DMZ) for public-facing services, and an isolated Internal network segment for sensitive internal resources. A Linux-based router enforces segmentation between these tiers using iptables firewall rules.

This document serves as a reference for the defensive team, the penetration testing team, and the grading assessors. It provides complete visibility into the network topology, host configurations, security controls, and the deliberate vulnerabilities introduced at each layer of the stack.

## 2. NETWORK TOPOLOGY

2.1 Topology Diagram

The following diagram illustrates the complete network topology of the vulnerable lab environment, including all hosts, network segments, IP addressing, and traffic flow paths:

Figure 1 — Network Topology Diagram — ACME Solution Industry Vulnerable Lab

2.2 Network Segment Overview

The environment is divided into three distinct network segments enforced at the hypervisor level using VMware LAN Segments (not Host-only networks — LAN Segments provide genuine isolation by excluding the host machine's routing table from participating in inter-segment communication):

| Segment | Type | VMware Network | Subnet | Gateway | Purpose |

| --- | --- | --- | --- | --- | --- |

| WAN / External | NAT | VMnet8 (NAT) | 192.168.64.0/24 | 192.168.64.2 (VMware NAT) | Simulates external internet — Parrot attacker and Router WAN interface |

| DMZ | Isolated | LAN Segment — DMZ | 192.168.10.0/24 | 192.168.10.2 (Router) | Public-facing service tier — DMZ Web Server only |

| Internal | Isolated | LAN Segment — Internal | 192.168.20.0/24 | 192.168.20.2 (Router) | Protected internal tier — Internal Host only |

VMware LAN Segments were selected over Host-only networks specifically because Host-only networks expose routes on the Windows host machine, allowing inter-segment communication that bypasses the router VM entirely. LAN Segments provide true hypervisor-level isolation.

2.3 Traffic Flow

All inter-segment traffic flows exclusively through the Router VM. The following traffic paths are permitted by the iptables FORWARD chain:

| Source | Destination | Protocol/Port | Purpose | Direction |

| --- | --- | --- | --- | --- |

| Parrot (192.168.64.136) | Router WAN (192.168.64.130:80) | TCP/80 | HTTP access to DVWA via DNAT | Inbound → |

| Router (iptables DNAT) | DMZ-Web (192.168.10.3:80) | TCP/80 | Forwarded web traffic to DVWA | → DMZ |

| DMZ-Web (192.168.10.3) | Internal-Host (192.168.20.3) | TCP/22 | SSH lateral movement (deliberate) | → Internal |

| DMZ-Web (192.168.10.3) | Parrot (192.168.64.0/24) | TCP/4444 | Reverse shell callback | → WAN |

| Internal-Host (192.168.20.3) | Internet | TCP/443, UDP/53 | Outbound HTTPS and DNS | → WAN |

| ALL | ALL | ALL others | Denied by default DROP rule | ✗ BLOCKED |

## 3. HOST INVENTORY

| Hostname | Role | OS | IP Address(es) | Hypervisor | NICs |

| --- | --- | --- | --- | --- | --- |

| routervm | Firewall / Router | Ubuntu Server 24.04 LTS | 192.168.64.130 (WAN)<br>192.168.10.2 (DMZ)<br>192.168.20.2 (Internal) | VMware Workstation | 3x NICs (ens33, ens37, ens38) |

| DMZtarget | DMZ Web Server | Ubuntu Server 24.04 LTS | 192.168.10.3 | VMware Workstation | 1x NIC — LAN Segment DMZ |

| manager | Internal Host | Ubuntu Server 24.04 LTS | 192.168.20.3 | VMware Workstation | 1x NIC — LAN Segment Internal |

| Parrot | Attacker Platform | Parrot OS Security Edition | 192.168.64.136 | VMware Workstation | 1x NIC — VMnet8 NAT |

## 4. HOST CONFIGURATION DETAIL

4.1 Router VM (routervm)

4.1.1 Role and Purpose

The Router VM acts as the sole boundary between all three network segments. It performs three distinct functions: IP packet forwarding between segments, Destination NAT (DNAT) to expose the DMZ web server through the WAN interface, and source address rewriting (MASQUERADE) to ensure return traffic is correctly routed back through the router rather than being dropped.

4.1.2 Network Interface Configuration

| Interface | VMware Network | IP Address | Role |

| --- | --- | --- | --- |

| ens33 | VMnet8 (NAT) | 192.168.64.130/24 (DHCP) | WAN-facing — Parrot attacker side |

| ens37 | LAN Segment DMZ | 192.168.10.2/24 (Static) | DMZ gateway — DMZ-Web default gateway |

| ens38 | LAN Segment Internal | 192.168.20.2/24 (Static) | Internal gateway — Internal-Host default gateway |

4.1.3 Netplan Static IP Configuration

Static IP addresses are configured via systemd-networkd using the following netplan configuration (/etc/netplan/01-static.yaml):

network:

version: 2

renderer: networkd

ethernets:

ens33:

dhcp4: true        # WAN side — DHCP from VMware NAT

ens37:

dhcp4: false

addresses:

- 192.168.10.2/24   # DMZ gateway

ens38:

dhcp4: false

addresses:

- 192.168.20.2/24   # Internal gateway

4.1.4 iptables Firewall Rule Set

All firewall rules are loaded via a startup script (/home/manager/router_startup.sh) executed at boot via cron @reboot, ensuring rules persist across reboots:

| Rule # | Chain | In IF | Out IF | Source | Destination | Proto/Port | Action | Purpose |

| --- | --- | --- | --- | --- | --- | --- | --- | --- |

| 1 | FORWARD | * | * | 0.0.0.0/0 | 0.0.0.0/0 | ESTABLISHED/RELATED | ACCEPT | Allow return traffic for existing sessions |

| 2 | FORWARD | ens33 | ens37 | 0.0.0.0/0 | 192.168.10.3 | TCP/80 | ACCEPT | Parrot → DMZ-Web HTTP |

| 3 | FORWARD | ens37 | ens38 | 192.168.10.3 | 192.168.20.3 | TCP/22 | ACCEPT | DMZ-Web → Internal SSH pivot |

| 4 | FORWARD | ens37 | ens33 | 192.168.10.3 | 192.168.64.0/24 | TCP/4444 | ACCEPT | Reverse shell callback |

| 5 | FORWARD | ens38 | ens33 | 192.168.20.3 | 0.0.0.0/0 | TCP/443 | ACCEPT | Internal HTTPS egress |

| 6 | FORWARD | ens38 | ens33 | 192.168.20.3 | 0.0.0.0/0 | UDP/53 | ACCEPT | Internal DNS egress |

| 7 | FORWARD | ens33 | ens37 | 0.0.0.0/0 | 192.168.10.3 | TCP/22 | ACCEPT | Management SSH to DMZ |

| 8 | FORWARD | ens33 | ens38 | 0.0.0.0/0 | 192.168.20.3 | TCP/22 | ACCEPT | Management SSH to Internal |

| 9 | FORWARD | * | * | 0.0.0.0/0 | 0.0.0.0/0 | ALL | DROP | Default deny — all other traffic |

4.1.5 NAT Rules

| Table | Chain | Rule | Purpose |

| --- | --- | --- | --- |

| nat | PREROUTING | -i ens33 -p tcp --dport 80 -j DNAT --to-destination 192.168.10.3:80 | Rewrites destination of incoming port-80 traffic to DMZ-Web — makes DVWA accessible via router WAN IP |

| nat | POSTROUTING | -o ens37 -j MASQUERADE | Rewrites source IP for DMZ-bound traffic — ensures DMZ-Web replies route back through router |

| nat | POSTROUTING | -o ens33 -j MASQUERADE | Rewrites source IP for WAN-bound traffic — ensures reverse shell replies reach Parrot correctly |

IP forwarding is enabled persistently via net.ipv4.ip_forward=1 in /etc/sysctl.conf. Without this, the Linux kernel discards packets destined for other interfaces rather than forwarding them.

4.2 DMZ Web Server (DMZtarget)

4.2.1 Role and Purpose

DMZtarget is the public-facing entry point of the vulnerable network. It hosts DVWA (Damn Vulnerable Web Application) v2.5 on Apache 2.4.58, simulating a poorly secured, externally-accessible web application. It is reachable from Parrot only through the router's DNAT rule — it has no direct WAN-side interface.

4.2.2 Network Interface Configuration

| Interface | VMware Network | IP Address | Gateway | Type |

| --- | --- | --- | --- | --- |

| ens37 | LAN Segment DMZ | 192.168.10.3/24 (Static) | 192.168.10.2 (Router) | Sole network interface |

4.2.3 Software Stack

| Component | Version | Purpose | Configuration |

| --- | --- | --- | --- |

| Ubuntu Server | 24.04 LTS | Operating system | Minimal install, systemd-networkd |

| Apache HTTP Server | 2.4.58 | Web server | Listening on port 80, Indexes enabled (deliberate) |

| PHP | 8.3 | Server-side scripting | libapache2-mod-php installed, executes DVWA |

| DVWA | 2.5 | Vulnerable web application | Security level: LOW, database initialised |

| OpenSSH | 9.6p1 | Remote access | Running on port 22 (deliberately exposed on DMZ) |

| MySQL | 8.0 | Database backend | DVWA database configured with default credentials |

4.2.4 Deliberately Introduced Vulnerabilities

| Vulnerability | Location | Severity | Description |

| --- | --- | --- | --- |

| OS Command Injection | DVWA Command Injection module | CRITICAL | User input passed directly to shell_exec() with no sanitisation at LOW security level |

| Default Credentials | DVWA login | HIGH | Application accessible with default admin/password credentials |

| Unsecured SSH Private Key | /var/www/.ssh/id_rsa | CRITICAL | RSA private key owned by www-data, no passphrase, grants SSH access to Internal-Host |

| SSH Exposed on DMZ | Port 22 open externally | HIGH | SSH service should not be accessible on a DMZ-facing host |

| Apache Directory Listing | Apache Options: Indexes | MEDIUM | Directory contents of /var/www/html/ exposed to unauthenticated users |

| Apache Version Disclosure | Server HTTP header | LOW | Full version string Apache/2.4.58 (Ubuntu) broadcast in all responses |

4.3 Internal Host (manager)

4.3.1 Role and Purpose

The internal host simulates a protected internal server accessible only via lateral movement from the DMZ web server. It contains a privilege escalation vulnerability via a misconfigured sudoers entry, and hosts planted cloud credentials used for the final exfiltration step. It is not reachable directly from the WAN — only via SSH from DMZtarget through the router's FORWARD rule.

4.3.2 Network Interface Configuration

| Interface | VMware Network | IP Address | Gateway | Type |

| --- | --- | --- | --- | --- |

| ens33 | LAN Segment Internal | 192.168.20.3/24 (Static) | 192.168.20.2 (Router) | Primary network interface |

| ens37 | VMnet8 NAT | 192.168.64.x (DHCP) | N/A (use-routes: false) | Internet access only — no default gateway pushed |

The NAT interface on the Internal Host is configured with dhcp4-overrides: use-routes: false in netplan to prevent VMware's NAT gateway from overriding the router as the default gateway. This maintains correct inter-subnet routing while still allowing outbound internet access via VMware's NAT infrastructure for the cloud exfiltration step.

4.3.3 User Accounts

| Username | Type | Shell | Purpose | Sudo Rights |

| --- | --- | --- | --- | --- |

| root | System | /bin/bash | Root account — target for privilege escalation | Full (implicit) |

| manager | Admin | /bin/bash | Management account used during lab build | Full via sudo |

| devadmin | Low-privilege | /bin/bash | Simulated developer account — attack target | NOPASSWD: /usr/bin/vim only (deliberate misconfiguration) |

4.3.4 Deliberately Introduced Vulnerabilities

| Vulnerability | Location | Severity | Description |

| --- | --- | --- | --- |

| Sudo NOPASSWD — vim | /etc/sudoers | CRITICAL | devadmin ALL=(ALL) NOPASSWD: /usr/bin/vim — vim shell escape grants immediate root |

| SSH Key Authentication | /home/devadmin/.ssh/authorized_keys | CRITICAL | Public key installed granting password-free access to attacker holding DMZtarget's private key |

| Cloud Credentials on Disk | /root/.aws/credentials | CRITICAL | AWS/Oracle Cloud API keys stored in plaintext in root's home directory |

| Unrestricted Egress | Router iptables rule 5 | MEDIUM | Outbound HTTPS from internal host not restricted to approved destinations — enables data exfiltration |

4.4 Attacker Platform (Parrot)

4.4.1 Role and Purpose

The Parrot OS instance simulates an external threat actor. It has no special privileges or prior knowledge of the target environment. It sits on the WAN-side NAT network and can only reach the target infrastructure through the router's WAN IP address (192.168.64.130).

4.4.2 Network Interface Configuration

| Interface | VMware Network | IP Address | Gateway |

| --- | --- | --- | --- |

| ens33 | VMnet8 NAT | 192.168.64.136/24 (DHCP) | 192.168.64.2 (VMware NAT) |

4.4.3 Tools Available

| Tool | Purpose | Usage in This Assessment |

| --- | --- | --- |

| nmap | Network scanning and service enumeration | Initial reconnaissance — port/service/version discovery on router WAN IP |

| gobuster | Web directory enumeration | Discovery of /DVWA/ path on target web server |

| Firefox | Web browser | DVWA interaction, command injection delivery |

| netcat (nc) | TCP listener and connectivity testing | Reverse shell listener on port 4444 |

| curl | HTTP/HTTPS request tool | Testing web connectivity, cloud bucket exfiltration |

| SSH client | Secure shell | Management access via ProxyJump through router |

| python3 | Scripting | PTY spawn for shell upgrade post-exploitation |

## 5. CLOUD INFRASTRUCTURE

5.1 Oracle Cloud Object Storage

An Oracle Cloud Infrastructure (OCI) Object Storage bucket was provisioned as the final target in the attack chain, simulating a misconfigured cloud storage resource containing sensitive organisational data. The bucket represents the exfiltration destination — data that an attacker with root access to the internal host and discovered cloud credentials can access and download.

5.2 Bucket Configuration

| Setting | Configuration | Security Status |

| --- | --- | --- |

| Visibility | Public | MISCONFIGURED — should be Private |

| Access Control | No authentication required for read | MISCONFIGURED — unauthenticated access enabled |

| Encryption at Rest | Default OCI-managed | Acceptable — does not prevent access once bucket is public |

| Access Logging | Disabled | MISCONFIGURED — no audit trail of access |

| Versioning | Disabled | Not configured — no object history |

| Data Contents | Seeded sensitive data (fake customer CSV, internal credentials) | CRITICAL EXPOSURE |

5.3 Access Chain

The cloud infrastructure is accessed via the following chain during the penetration test:

Attacker achieves root on Internal-Host (192.168.20.3) via the privilege escalation chain

Attacker discovers /root/.aws/credentials containing Oracle Cloud API keys

Internal-Host's outbound HTTPS rule (router FORWARD rule 5) permits TCP/443 to the internet

Attacker uses curl to access the publicly-readable bucket endpoint directly — no API keys strictly required due to public-read misconfiguration

Sensitive data is exfiltrated to /tmp/ on the Internal-Host and confirmed downloaded

## 6. INTENDED ATTACK CHAIN

The network was designed to support a specific, chained attack path from initial external reconnaissance through to cloud data exfiltration. Each vulnerability in the chain depends on the previous step, creating a realistic multi-stage compromise scenario:

| Step | Stage | From | To | Technique | Vulnerability Exploited |

| --- | --- | --- | --- | --- | --- |

| 1 | Reconnaissance | Parrot (External) | Router WAN IP | nmap -sV -sC | None — passive scanning |

| 2 | Initial Access | Parrot | DMZtarget via router DNAT | OS Command Injection | DVWA Command Injection (LOW security) |

| 3 | Discovery | DMZtarget (www-data shell) | Local filesystem | find /var/www -name id_rsa | Unsecured SSH private key |

| 4 | Lateral Movement | DMZtarget (www-data) | manager (Internal-Host) | SSH with discovered key | Unprotected private key + authorized_keys |

| 5 | Privilege Escalation | manager (devadmin) | manager (root) | sudo vim -c ':!/bin/bash' | sudo NOPASSWD vim misconfiguration |

| 6 | Discovery | manager (root) | Local filesystem | cat /root/.aws/credentials | Cloud credentials stored in plaintext |

| 7 | Exfiltration | manager (root) | Oracle Cloud Bucket | curl https://objectstorage... | Public cloud bucket + unrestricted egress |

## 7. SECURITY CONTROLS

The following security controls are implemented in the environment. Note that while these controls are present, the deliberately introduced vulnerabilities provide clear bypass paths for each:

| Control | Implementation | Bypass in This Lab |

| --- | --- | --- |

| Network Segmentation | LAN Segments enforcing DMZ/Internal isolation at hypervisor level | Router's SSH FORWARD rule (rule 3) permits DMZ-to-Internal pivot via discovered SSH key |

| Firewall (iptables) | Default-deny FORWARD policy with explicit allow rules | Reverse shell uses permitted port 4444 outbound; SSH pivot uses permitted port 22 |

| DNAT / NAT | DMZ-Web not directly reachable — accessed only via router DNAT | Attacker reaches DVWA via DNAT as intended; same rule enables exploitation |

| MASQUERADE | Source IP rewriting hides internal addressing from WAN | Does not prevent exploitation — functional security measure only |

| SSH Key Authentication | Password authentication not the primary vector | Key discovered on DMZ-Web filesystem — bypassed by T1552.004 |

| User Account Separation | devadmin lacks general sudo rights | NOPASSWD vim entry provides trivial privesc — GTFOBins T1548.003 |

## 8. MANAGEMENT ACCESS

8.1 Administrative SSH Access Pattern

Administrative access to DMZ-Web and Internal-Host is provided via a bastion/jump host pattern through the Router VM. The router is the only VM with a NAT-side interface accessible from the Windows host machine. All management connections to isolated VMs are proxied through it:

# Windows host ~/.ssh/config

**Host router**

HostName 192.168.64.130

**User manager**

**Host dmz-web**

HostName 192.168.10.3

User <admin-user>

**ProxyJump router**

**Host internal-host**

HostName 192.168.20.3

**User devadmin**

**ProxyJump router**

IdentityFile ~/.ssh/lab_internal_id_rsa

This management pattern reflects real-world enterprise bastion host architecture. In production, the bastion host function and the router/firewall function would be separated onto dedicated infrastructure to prevent a compromised jump host from also being the network firewall.

## 9. KNOWN ISSUES & DESIGN DECISIONS

| Issue | Root Cause | Decision / Workaround |

| --- | --- | --- |

| Router WAN IP is DHCP | VMnet8 NAT assigns IPs dynamically — static not possible without VMware config changes | Accepted — WAN IP floats; iptables rules use interface names not IPs; payloads must reference current WAN IP |

| Docker installed on all VMs | Pre-existing Ubuntu template had Docker installed | Disabled permanently via systemctl disable docker on all VMs — Docker hijacks iptables nat table which breaks custom routing rules |

| iptables rules lost on reboot | netfilter-persistent unreliable when switching from NetworkManager to networkd | Replaced with cron @reboot startup script (/home/manager/router_startup.sh) that reloads all rules on every boot |

| Internal-Host dual NIC | NAT NIC added for internet access (cloud exfil step) | dhcp4-overrides: use-routes: false prevents NAT gateway from overriding router as default — both internet and inter-subnet routing work correctly |

| VMware Host-only initial design | Initially used Host-only VMnets — Windows host machine has routes to all Host-only subnets, bypassing router | Switched to LAN Segments — truly isolated, host machine has no routes to these networks |

## 10. DOCUMENT SIGN-OFF

| Role | Name | Signature | Date |

| --- | --- | --- | --- |

| Cybersecurity Lead | Ejiofor Godswill | ________________________ | 8 July 2026 |

| Cloud Engineering Lead | Demilada | ________________________ | 8 July 2026 |

| Assessor / Grader | ________________________ | ________________________ | ________________________ |

ACME Solution Industry — Network Architecture Document — CONFIDENTIAL — July 2026
