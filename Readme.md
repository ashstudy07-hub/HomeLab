### HomeLab description

![Alt Text](/CyberSecHomeLAB2.drawio.png)

This is a SOC Home Lab designed to run various SOC tools on a local server.
The diagram above shows the high level layout of the LAB.

All the VMs are being hosted on a Dell PowerSedge R630 server using Proxmox Virtual Environment.
The Dell server's builtin iDRAC feature is used as the management interface.

For the purpose of this lab, the server is connected to the Home network [192.168.1.1] via a Bridge Router. 

For convenience a seperate Management PC which is connected to the home network will be used. 

A pfSense firewall is setup to monitor traffic and to handle network configurations.

Within the Proxmox server, the virtual machines are seperated into the following subnet/VLANS:

| Name | Subnet/VLAN | Purpose |
|-------------|-----------| -------|
|MGMT|10.10.1.0/24| SOC tools |
|LAN|10.10.10.0/24| End users |
|DMZ-ATTACK VLAN20|10.10.20.0/24| Attack machine to simulate various network scans and attacks |
|DMZ-TARGET VLAN30|10.10.30.0/24| Vulnerable machines for attacker to target |

