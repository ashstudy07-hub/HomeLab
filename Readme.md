### HomeLab description

![Alt Text](/CyberSecHomeLAB2.drawio.png)

This is a SOC Home Lab designed to run various SOC tools on a local server.
The diagram above shows the high-level layout of the LAB.

All the VMs are being hosted on a Dell PowerSedge R630 server using Proxmox Virtual Environment.
The Dell server's built-in iDRAC feature is used as the management interface.

For this lab, the server is connected to the Home network [192.168.1.1] via a Bridge Router. 

For convenience, a separate Management PC which is connected to the home network will be used. 

A pfSense firewall is deployed to monitor traffic and to handle network configurations.

Within the Proxmox server, the virtual machines are separated into the following subnet/VLANS:

| Name | Subnet/VLAN | Purpose |
|-------------|-----------| -------|
|MGMT|10.10.1.0/24| SOC tools |
|LAN|10.10.10.0/24| End users |
|DMZ-ATTACK VLAN20|10.10.20.0/24| Attack machine to simulate various network scans and attacks |
|DMZ-TARGET VLAN30|10.10.30.0/24| Vulnerable machines for attacker to target |

## SOC TOOLS 

### Splunk Security information and event management (SIEM).
Splunk is used for centralised log management, real-time threat detection and provides security teams with a unified view to identify, investigate, and respond to security incidents.

This document outlines the initial setup and configuration process for a SPLUNK deployment. [READ HERE](/splunk-setup-config.md)

### Wazuh XDR/SIEM
Wazuh is a free, open-source security platform that unifies XDR (Extended Detection and Response) and SIEM (Security Information and Event Management) capabilities.
Wazuh has several features that help monitor endpoint devices, check compliance, detect vulnerabilities and respond to incidents.

This document outlines the initial setup and configuration process for a Wazuh deployment. [READ HERE](/Wazuh-setup-config.md)

One of the features that Wazuh provide is File Integrity Monitor (FIM), which allows us to monitor activity on the specified folder on the end host. This feature can be used with [VirusTotal integration](/Wazuh-VirusTotal-Integration.md) for monitoring and inspecting monitored files for malicious content.

Suricata is used as an open-source network intrusion detection system (IDS) and intrusion prevention system (IPS) to monitor network traffic for malicious activity. As we have a pfsense firewall already setup for our Lab, we can run [SURICATA](/Suricata-on-pfsense.md) on pfSense as a service. 
Instead of sending the logs from Suricata to SPLUNK and WAZUH directly (which can sometimes result in only one SIEM receiving the logs), we will be deploying a separate log collector VM that receives the logs from Suricata and forwards them to the two SIEMs. 

Since we are using the Management PC to access and manage all the VMs, the traffic has to pass through the pfSense firewall. Due to the nature of our lab setup, this will create asymemetric traffic, and the firewall will drop the connection. The resolution to this is using [NAT Port Forwarding](/portforwarding-on-pfsense.md).