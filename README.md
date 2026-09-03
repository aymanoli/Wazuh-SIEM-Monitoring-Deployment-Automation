# SIEM Deployment & SOC Automation Pipeline (Wazuh, TheHive, Shuffle)

## Overview
This project demonstrates a Wazuh SIEM home lab built for hands-on SOC Analyst training. It showcases endpoint security monitoring, log analysis, File Integrity Monitoring (FIM), vulnerability detection, custom rule creation, and brute-force attack detection using a Windows endpoint and Ubuntu-based Wazuh Manager. The lab simulates real-world security events to practice alert validation, investigation, and threat detection.

## Lab Architecture
| Components  | System  |
|-------------|---------|
| Wazuh Server | Ubuntu Server |
| Wazuh Agent  | Windows Agent |
| Wazuh Agent  | Ubuntu Agent |

## Project Overview
* Deployed and configured a distributed security stack across three VMs Wazuh agent/manager for endpoint detection and monitoring, and a TheHive/Cassandra/Elasticsearch cluster for centralized incident response.
* Configured File Integrity Monitoring (FIM), Windows event log collection, vulnerability detection, and MITRE ATT&CK mapped alert visibility within the Wazuh dashboard.
* Implemented a custom Wazuh rule (local_rules.xml) to detect brute-force authentication attempts and configured Active Response to automatically block SSH brute-force attacks using a second Ubuntu agent.
* Designed and automated a SOAR workflow in Shuffle (Webhook → SHA256 Hash Extraction → VirusTotal → TheHive Alert Creation → Email Notification), including securely exposing the on-prem TheHive instance via Cloudflare Tunnel to integrate with the cloud-hosted runtime over HTTPS without inbound port forwarding.
* Validated the pipeline end-to-end using a live Mimikatz execution on the Windows endpoint detection, VirusTotal hash enrichment, automatic TheHive alert creation, and real-time email notification.

## Implemented Features
* Wazuh server installation and configuration in Ubuntu
* Wazuh agents deployed and connected in Windows
* Create custom rules to identify a brute-force attack
* File Integrity Monitoring (FIM)
* Windows endpoint monitoring
* Security event collection
* Basic alert investigation
* MITRE ATT&CK mapping
* SIEM Automations

## ScreenShots
#### SIEM Automation
<img width="1920" height="939" alt="Screenshot (1446)" src="https://github.com/user-attachments/assets/907c32d0-4bc6-466b-a29a-9d0483c082ab" />
<img width="1920" height="532" alt="Screenshot (1443)" src="https://github.com/user-attachments/assets/adfbb71a-1140-422e-b5bc-16a5b4d76f0a" />

#### Windows Agent Overview
<img width="1920" height="1080" alt="Screenshot From 2026-07-11 22-29-53" src="https://github.com/user-attachments/assets/d454a447-65c1-4e54-98c8-13d08f16eb08" />

### Windows event log collection
<img width="1920" height="955" alt="Screenshot From 2026-07-11 10-08-45" src="https://github.com/user-attachments/assets/9afe498a-5b96-4c7b-a65b-e4e622e62201" />

### File Integrity Monitoring
<img width="1920" height="1080" alt="Screenshot From 2026-07-11 22-33-12" src="https://github.com/user-attachments/assets/2580e7fc-df2f-4bba-aaa1-dce49518a0dd" />

### MITRE ATT&CK mapped
<img width="1920" height="1080" alt="Screenshot From 2026-07-11 22-32-10" src="https://github.com/user-attachments/assets/4962f535-1157-4bf1-83fe-c90e483c42e4" />

### Vulnerability detection
<img width="1920" height="1080" alt="Screenshot From 2026-07-11 22-54-13" src="https://github.com/user-attachments/assets/ecab57c7-129c-4855-83b2-a98e8afdea37" />


