# Web Server Exploitation & SIEM Detection

## Overview

This project demonstrates the detection and forensic investigation of a controlled multi-stage web server compromise using Splunk SIEM.

A deliberately vulnerable web application was deployed in an isolated academic lab environment using Ubuntu Server, Apache2, Flask and MySQL. A controlled attack was then performed against the environment, followed by SIEM-based analysis of the resulting security logs.

The main blue-team objective was to compare a clean pre-attack baseline against post-attack evidence and reconstruct the attack from multiple log sources.

> This was completed as a collaborative university group project. This repository presents a sanitized SOC-focused summary of the investigation rather than the original assessment submission.

## Technologies

- Ubuntu 22.04 Server
- Apache2
- Flask
- MySQL
- Splunk Enterprise
- Splunk Universal Forwarder
- Kali Linux
- SQLMap
- SSH
- Linux authentication logs
- File-integrity hashing

## Lab Scenario

The lab simulated a multi-stage compromise beginning with a vulnerable Flask login application.

The attack progressed through:

1. SQL injection
2. Database enumeration
3. Credential extraction
4. Authentication bypass
5. Unauthorized SSH access
6. Data exfiltration
7. File tampering
8. Privilege escalation
9. Access to previously restricted information

All activity was carried out in an isolated academic environment.

## Detection Architecture

The Ubuntu server generated security-relevant telemetry while Splunk Enterprise provided centralized log collection and analysis.

A Splunk Universal Forwarder on the Ubuntu server forwarded security data to Splunk Enterprise running on the Windows host.

The investigation used evidence from:

- Web/application access logs
- Linux authentication logs
- MySQL logs
- File-integrity hashes

A clean baseline was captured before the attack so that legitimate system activity could be compared against post-compromise behaviour.

## SIEM Verification

Log ingestion was first validated in Splunk using:

```
index=* | head 20
```
During the project, 269 events were collected across the monitored log sources.

This confirmed that Splunk was receiving sufficient telemetry to reconstruct activity across the attack lifecycle.

## Detection 1 — SQL Injection Activity

The initial compromise involved automated SQL injection against the vulnerable Flask login page.

The application logs showed:

- a large number of POST requests within a short period;
- requests originating from the attacking system;
- a visible SQLMap user-agent string;
- abnormal request volume compared with the clean baseline.

From a SOC perspective, the combination of high-frequency POST activity and an identifiable automated-tool user agent would be a strong investigation indicator.

### SOC Interpretation

A high volume of unusual requests from a single source should be correlated with:

- the affected application;
- source IP;
- request frequency;
- user-agent information;
- authentication behaviour;
- subsequent activity on the server.

The alert should not be treated as an isolated web event if later authentication or host activity is associated with the same attack timeline.

## Detection 2 — Unauthorized SSH Access

After credentials were exposed, an SSH session was established using the low-privileged `remote_admin` account.

The authentication activity was investigated in Splunk using:

```
index=* sourcetype=syslog "remote_admin"
```

The search returned events showing a successful password authentication and SSH session creation.

This provided evidence including:

- source IP;
- affected account;
- authentication method;
- timestamp;
- successful session establishment.

### SOC Interpretation

A successful SSH login becomes more significant when it follows suspicious web activity or credential exposure.

An analyst should correlate:

```
Web exploitation
        ↓
Credential exposure
        ↓
Successful SSH authentication
        ↓
Post-login activity
```

rather than assessing the SSH event independently.

## Detection 3 — File Exfiltration and Tampering

The lab contained dummy confidential files with deliberately different Linux permissions.

During the compromise:

- files accessible to the low-privileged account were exfiltrated;
- a world-writable file was modified;
- more restricted files initially resisted modification.

MD5 hashes had been recorded before the attack.

The hashes were recalculated afterward and compared with the baseline, providing evidence that file contents had changed.

### SOC Interpretation

File-integrity changes involving sensitive files should be investigated alongside:

- the user performing the action;
- recent authentication activity;
- privilege level;
- file permissions;
- surrounding process and command activity.

## Detection 4 — Privilege Escalation

A sudo configuration allowed the low-privileged account to execute Python as root without requiring a password.

The resulting privileged activity was recorded in the authentication logs.

This allowed the investigation to correlate:

```
Low-privilege SSH session
        ↓
Sudo enumeration
        ↓
Privileged command execution
        ↓
Root access
        ↓
Access to previously restricted data
```

Unexpected sudo activity following a suspicious login represents an important escalation indicator for a SOC analyst.

## Full Attack Timeline

A chronological view was created in Splunk using:

```
index=* | sort _time | table _time sourcetype _raw
```

This produced a timeline covering the monitored events across the attack lifecycle.

The correlated evidence showed progression through:

```
SQL Injection
      ↓
Credential Exposure
      ↓
SSH Access
      ↓
Data Access / Exfiltration
      ↓
Privilege Escalation
      ↓
File Tampering
```

This was the most useful part of the project from a SOC perspective because it demonstrated how multiple individual log events can be combined to reconstruct a complete security incident.

## Detection Opportunities

Based on the investigation, useful SIEM detections could include:

- SQLMap or other suspicious automated user agents
- unusually high POST request frequency
- successful SSH access from unexpected sources
- abnormal sudo command execution
- use of privileged scripting interpreters
- changes to sensitive files
- unexpected changes to file-integrity hashes
- suspicious activity across multiple log sources within the same time window

## Recommended Defensive Actions

The investigation identified several defensive improvements:

- use parameterized queries to prevent SQL injection;
- enforce least-privilege sudo permissions;
- restrict sensitive file permissions;
- use SSH key-based authentication where appropriate;
- avoid exposing infrastructure credentials through applications;
- implement file-integrity monitoring;
- configure SIEM detections for suspicious web, authentication and privileged activity.

## Key SOC Lessons

This project helped demonstrate several principles relevant to Security Operations:

- a security alert should be investigated in context rather than in isolation;
- baselining makes abnormal behaviour easier to identify;
- centralized log collection allows analysts to correlate activity across different systems;
- successful authentication following suspicious activity can materially change the severity of an investigation;
- privilege escalation and file modifications can indicate progression of a compromise;
- timelines are valuable for understanding how far an attacker progressed.

## My Contribution

This was a collaborative university project.

My practical work included participating in the controlled attack scenario using SQL injection tooling and connecting the observed offensive activity with the later defensive detection and analysis.

The project helped me understand how attacker behaviour produces evidence that can later be investigated by a SOC analyst.

## Disclaimer

All attack activity was performed in a controlled academic lab environment using intentionally vulnerable systems and dummy data.

No unauthorized systems or real confidential data were targeted.
