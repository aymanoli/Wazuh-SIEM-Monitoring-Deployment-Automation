# SIEM Deployment & SOC Automation Pipeline
### Wazuh · TheHive · Shuffle SOAR · VirusTotal · Email · Cloudflare Tunnel
A fully self-hosted, end-to-end security operations lab: an endpoint detection triggers automated threat intelligence enrichment, incident case creation, and real-time analyst notification — with zero manual intervention.
<hr>

## Overview
This project simulates a real-world SOC (Security Operations Center) detection-to-response workflow using open-source tools deployed across an isolated multi-VM lab environment. A malicious credential-dumping tool (Mimikatz) executed on a monitored Windows endpoint triggers a chain of automated actions: hash extraction, threat intelligence lookup, case creation in an incident response platform, and email alerting to the analyst — all within seconds.


## Architecture
```
┌─────────────────────┐
│   Windows VM        │
│   (Wazuh Agent)     │  ← Mimikatz executed here (simulated attack)
└──────────┬──────────┘
           │ logs/events
           ▼
┌─────────────────────┐
│   Ubuntu VM         │
│   (Wazuh Manager)   │  ← Detects & generates alerts
└──────────┬──────────┘
           │ webhook trigger
           ▼
┌─────────────────────────────────────────────────────────────┐
│                    Shuffle SOAR (Cloud)                     │
│                                                             │
│  Webhook → SHA256 Hash Extraction → VirusTotal v3 Lookup    │
│                          │                                  │
│                          ▼                                  │
│              TheHive Alert Creation → Email Notification    │
└──────────────────────────┬──────────────────────────────────┘
                           │  HTTPS (via Cloudflare Tunnel)
                           ▼
              ┌─────────────────────────┐
              │   Ubuntu VM             │
              │   TheHive + Cassandra   │
              │   + Elasticsearch       │
              │   (Incident Response)   │
              └─────────────────────────┘
```
<img width="1920" height="939" alt="Screenshot (1446)" src="https://github.com/user-attachments/assets/b59900c7-ebaa-4c26-8ff1-caa069c30eac" />

**Lab environment:** 3 isolated VirtualBox VMs on a single host:
1. **Windows 10** — Wazuh agent + Mimikatz (attack simulation endpoint)
2. **Ubuntu** — Wazuh manager (centralized log collection & detection)
3. **Ubuntu** — TheHive + Cassandra + Elasticsearch + Cloudflare Tunnel (incident response platform)

## Tech Stack

| Category | Tool |
|---|---|
| Endpoint Detection | Wazuh (Agent + Manager) |
| Incident Response Platform | TheHive 5 |
| Graph Database | Cassandra (JanusGraph backend) |
| Search & Indexing | Elasticsearch |
| SOAR / Automation | Shuffle (Cloud runtime) |
| Threat Intelligence | VirusTotal API v3 |
| Secure Tunneling | Cloudflare Tunnel |
| Notification | Email (SMTP via Shuffle) |
| Simulated Threat | Mimikatz |
| Virtualization | Oracle VirtualBox |
| OS | Ubuntu Server, Windows 10 |

## Workflow Breakdown

The core automation, built in Shuffle:

```
Webhook  →  SHA256 Hash Extraction  →  VirusTotal v3 Lookup  →  TheHive Alert Creation  →  Email Notification
```

1. **Webhook** — receives the triggering event/hash from the detection source
2. **SHA256 Hash Extraction** — parses the file hash from the incoming payload
3. **VirusTotal v3 Lookup** — queries VT's database for known malicious indicators tied to the hash
4. **TheHive Alert Creation** — automatically opens a formal incident alert in TheHive with enrichment data attached
5. **Email Notification** — sends a real-time alert to the analyst's inbox

## Automated Incident Response Backend: TheHive + Cassandra + Elasticsearch
TheHive was deployed on Ubuntu with Cassandra as the graph database (via JanusGraph) and Elasticsearch as the search/index engine, all configured to communicate over plain HTTP on the local network.
**What is changed in all the configuration files**
```hocon
# sudo nano /etc/thehive/application.conf
db.janusgraph {
  storage {
    backend = cql
    hostname = ["192.168.1.6"]
    cql {
      cluster-name = test-thehive
      keyspace = thehive
    }
  }
  index.search {
    backend = elasticsearch
    hostname = ["192.168.1.6"]
    index-name = thehive
    ssl {
      enabled = false
    }
  }
}
application.baseUrl = "http://192.168.1.6:9000"
```
```hocon
# sudo nano /etc/cassandra/cassandra.yml
cluster_name: 'test-thehive'
listen_address: 192.168.1.6
rpc_address: 192.168.1.6
- seeds: "192.168.1.6:7000"
```
```hocon
# sudo nano /etc/elasticsearch/elasticsearch.yml
cluster.name: myself
node.name: node-1
network.host: 192.168.1.6
http.port: 9200
cluster.initial_master_nodes: ["node-1"]
```
Cassandra (`cassandra.yaml`) and Elasticsearch (`elasticsearch.yml`) were configured with matching cluster names and network bindings, and Elasticsearch's `xpack.security` was disabled to match TheHive's plain-HTTP configuration.

## Exposing TheHive Securely: Cloudflare Tunnel
The Shuffle Cloud runtime needed to reach TheHive, but TheHive only had a private LAN IP (`192.168.1.6`). **Port forwarding was not usable in this case, because the ISP assigns a private (CGNAT) IP address rather than a public one** — so no inbound port can ever reach the home router. Instead, **Cloudflare Tunnel** was used, which only requires an outbound connection and works without any port forwarding or public IP.

**Setup:**

```bash
# 1. Install cloudflared on the TheHive VM
sudo apt install ./cloudflared-linux-amd64.deb

# 2. Confirm TheHive is reachable locally first
curl http://localhost:9000

# 3. Create a systemd service to run the tunnel persistently
sudo nano /etc/systemd/system/cloudflared-quick.service
```

```ini
[Unit]
Description=Cloudflare Quick Tunnel for TheHive
After=network.target

[Service]
ExecStart=/usr/local/bin/cloudflared tunnel --url http://localhost:9000
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```

```bash
# 4. Enable and start the tunnel
sudo systemctl daemon-reload
sudo systemctl enable cloudflared-quick
sudo systemctl start cloudflared-quick

# 5. Retrieve the generated public HTTPS URL
sudo journalctl -u cloudflared-quick --no-pager | grep trycloudflare
```
This produced a public URL (e.g. `https://alberta-bidder-borough-con.trycloudflare.com`), which was set as TheHive's endpoint in Shuffle's authentication config — allowing the cloud-hosted workflow to securely reach the on-prem TheHive instance over HTTPS.

> **Note:** The free Quick Tunnel generates a new URL each time the service restarts for 1 minute (after that you will get an error message). For that time-limit error, I cannot get an alert message in thehive. For a permanent setup, a Cloudflare **Named Tunnel** on a custom domain is the recommended production fix, but it is paid.

## Validation

The full pipeline was tested end-to-end using a live attack simulation:

1. **Mimikatz** was executed on the Windows endpoint (MITRE ATT&CK: T1003)
2. Wazuh detected the activity and triggered the Shuffle webhook
3. The file hash was automatically extracted and queried against VirusTotal
4. An alert was created in TheHive, reachable through the Cloudflare Tunnel
5. A real-time email notification was received confirming the detection
✅ **Result:** alert successfully created in TheHive, and email notification delivered within seconds of execution.
<img width="1920" height="532" alt="Screenshot (1443)" src="https://github.com/user-attachments/assets/e3185c61-cec1-42f3-b6fb-ec00f06a0222" />


























