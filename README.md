# 🛡️ SOC Home Lab: Wazuh, Suricata & GNS3 Network
![Wazuh](https://img.shields.io/badge/SIEM-Wazuh-blue) ![Suricata](https://img.shields.io/badge/IDS-Suricata-red) ![GNS3](https://img.shields.io/badge/Network-GNS3-orange) ![Cisco](https://img.shields.io/badge/Infrastructure-Cisco-black)

## 📌 Project Summary
This project demonstrates a reusable virtual home network designed to simulate a real-world enterprise environment. The goal was to implement **Defense-in-Depth** by integrating network segmentation (VLANs), an Intrusion Detection System (Suricata), and an XDR/SIEM platform (Wazuh).

**Key Accomplishments:**
* Deployed a segmented network using **Cisco vIOS** (Router/Switch) and **ASAv Firewall**.
* Configured **Suricata** to sniff traffic via a SPAN port mirror.
* Ingested logs into **Wazuh** from the Router, IDS, and Endpoints.
* Validated security policies by simulating attacks (Nmap) and verifying detection.

---

## 🛠️ Technologies & Tools
* **Hypervisor & Emulation:** VMware Workstation Pro, GNS3
* **Network Infrastructure:** * Cisco ASAv Firewall (`asav992-32.qcow2`)
  * Cisco vIOS L2 Switch (`viosl2-adventerprisek9`)
  * Cisco IOU L3 Router (`i86bi-linux-l3-jk9s-15.0.1.bin`)
* **Security & Monitoring:** Wazuh (SIEM/XDR) on Amazon Linux 2023, Suricata (IDS) on Ubuntu Server
* **Endpoints:** Kali Linux (Attacker), Windows 10 (Victim)

---
## 🏗️ Network Architecture
The network was simulated using GNS3 integrated with VMware Workstation Pro.

![Network topology view](screenshots/25.11.08_network.topology.png)

### VLAN Configuration
I implemented network segmentation to isolate management traffic from endpoint traffic.

| VLAN ID | Name | Subnet | Description |
| :--- | :--- | :--- | :--- |
| **10** | Management | `192.168.10.0/24` | Wazuh Server & Suricata |
| **20** | Kali | `192.168.20.0/24` | Attacker Machine (Dept A) |
| **30** | Windows | `192.168.30.0/24` | Victim Machine (Dept B) |

*Cleaned Cisco Router and Switch configurations are available in the [/configs](configs) folder.*

---

## ⚔️ Attack & Defense Simulation
To validate the detection capabilities, I performed a live attack simulation.

### 1. The Attack
Using the Kali Linux VM (VLAN 20), I ran an aggressive Nmap scan against the Windows 10 endpoint (VLAN 30).
```bash
nmap -A 192.168.30.2
```
### 2. The Detection

Suricata detected the anomalous traffic via the SPAN port on the switch. The logs were forwarded to the Wazuh Manager, triggering the alert: **"ET SCAN Possible Nmap User-Agent Observed"**.

![Suricata alerts on nmap appearing in Wazuh](screenshots/25.11.09_nmap.wazuh.png)

---

## 🖧 Network Configuration Highlights

### 1. Inter-VLAN Routing (Router-on-a-Stick) 
I configured subinterfaces on the Core Router to allow traffic routing between the Management, Kali, and Windows VLANs while maintaining segmentation.
```Cisco CLI
! Router-on-a-Stick Configuration for VLAN 10 (Management)
interface Ethernet0/0.10
 description Gateway for Management VLAN
 encapsulation dot1q 10
 ip address 192.168.10.1 255.255.255.0
```
### 2. Traffic Analysis (SPAN Port)
To enable the Suricata IDS to inspect network traffic, I configured a Switched Port Analyzer (SPAN) session on the core switch. This mirrored all traffic from the router uplink to the monitoring interface.
```Cisco CLI
! Mirror traffic from the Router (Gi0/0) to Suricata (Gi2/2)
monitor session 1 source interface Gi0/0
monitor session 1 destination interface Gi2/2
```
### 3. Network Segmentation (ACLs)
I implemented Extended ACLs to enforce strict segmentation. The following configuration blocks the Windows 10 VLAN from accessing the Management subnet, while permitting specific ports (1514/1515) required for Wazuh agent communication.
```Cisco CLI
ip access-list extended SECURE-VLANS
 ! Permit Wazuh Agent Traffic (TCP/UDP)
 permit udp any host 192.168.10.10 eq 1514
 permit tcp any host 192.168.10.10 eq 1515
 ! Deny all other Windows VLAN traffic to Management VLAN
 deny ip 192.168.30.0 0.0.0.255 192.168.10.0 0.0.0.255
 permit ip any any
```
### 4. Centralized Logging (Syslog)
I configured the Core Router to forward all system logs and security events to the Wazuh SIEM for centralized monitoring and alerting.
```Cisco CLI
! Configure Router to send logs to Wazuh Server
logging host 192.168.10.10
logging trap informational
```

---

## 🔧 Challenges & Troubleshooting (Lessons Learned)

This project required significant troubleshooting. Here are the main technical hurdles I overcame:
### 1. Wazuh JSON Decoder Error

**Issue**: Suricata logs were not appearing in the Wazuh dashboard despite being forwarded. The logs showed `wazuh-analysisd ERROR Too many fields for JSON decoder`. 

**Solution**: The default decoder limit in Wazuh is 256 fields, but Suricata's `eve.json` is verbose. I modified the `internal_options.conf` file to increase the limit:
```bash
# /var/ossec/etc/internal_options.conf
analysisd.decoder_order_size=1024  # Changed from 256
```
I have since learned that custom changes should instead be made to the `/var/ossec/etc/local_internal_options.conf` file for them to persist through Wazuh upgrades.

### 2. Incompatible Switch Image (SPAN Port Failure)

**Issue**: While configuring the IDS, the `monitor session` command on the switch failed with an "Invalid Input" error. I discovered the initial Cisco IOU L2 image that I was using did not support Port Mirroring (SPAN), which is critical for sending traffic to Suricata. This was my largest setback during the project.

**Solution**: I migrated the network infrastructure to a **Cisco vIOS Layer 2** image (viosl2-adventerprisek9), which supports local SPAN. This required rebuilding the topology, and reentering all of the switch commands, but successfully enabled traffic inspection. In the future, I will try to verify the hardware capabilities of potential devices for my needs before implementing them.

### 3. Suricata Socket Permissions

**Issue**: The Suricata service repeatedly failed with `failed to create socket directory /var/run/suricata/: Permission denied`.

**Solution**: This was a permissions conflict on the Ubuntu server. I manually created the run directory and assigned ownership to the `suricata` user.

```bash
sudo mkdir /var/run/suricata
sudo chown suricata:suricata /var/run/suricata
```

---

## 📂 Configuration Files

[Router_config.cfg](configs/Router_config.cfg) - ACLs, Subinterfaces, and DHCP.

[Switch_config.cfg](configs/Switch_config.cfg) - VLANs, Trunking, and SPAN port.

[ossec.conf](configs/ossec.conf) - Wazuh Agent configuration snippets.

---

## ⚠️ Disclaimer
This repository and the associated lab environment are for educational purposes only. All attacks and simulations were conducted in an isolated, authorized, and virtualized environment to demonstrate defensive capabilities and network monitoring concepts.
