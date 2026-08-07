## SURICATA IDS/IPS ON pfSense firewall
Below is the current Homelab Setup:

```text

                    Internet
                        |
                   Home Router
                        |
                  192.168.1.0/24
                        |
                  pfSense Firewall
      -------------------------------------
      |         |             |           |
   MGMT      LAN          DMZ-20      DMZ-30
10.10.1.x 10.10.10.x   10.10.20.x   10.10.30.x
      |                    |             |
      |                  Kali      Metasploitable
      |
  Splunk      
  Wazuh

```

Suricata runs on pfSense, inspecting traffic on the LAN and both DMZ VLANs. pfSense then forwards Suricata logs to a log collector VM which then sends the log to Splunk and Wazuh.

# Phase 1 - Install Suricata on pfSense

## 1. Update pfSense

Go to

```
System → Update
```

Update to the latest version.

---

## 2. Install the package

```
System
    Package Manager
        Available Packages
```

Search for

```
Suricata
```

Install.

After installation you'll see
```
Services
    Suricata
```
# Phase 2 - Configure Interfaces

Open

```
Services
    Suricata
```

Select

```
Interfaces
```

Add the interfaces you want monitored.

I would add

```
LAN

DMZ (VLAN20)

DMZ_TARGET (VLAN30)

WAN (optional)
```

Avoid monitoring the MGMT interface unless you specifically want to inspect management traffic.

---

# Phase 3 - Configure Each Interface

*When dealing with multiple interfaces, it is good practice to setup one interface first and then copy the settings for the orther interfaces.*

Click the interface.

Enable

```
✔ Enable
```

EVE Output Settings

```
EVE JSON log ✔ Enable
EVE Output Type: SYSLOG
Tracked-Files Checksum: SHA256

Note: Leave the remaining setting as default
```

Initially avoid IPS until everything is working.


---

## Home Net

Set

```
HOME_NET

10.10.0.0/16
```

or

```
10.10.1.0/24
10.10.10.0/24
10.10.20.0/24
10.10.30.0/24
```

---

## External Net

```
!$HOME_NET
```
SAVE

EVE Output to syslog requires Suricata alerts to be copied to the system log, so 'Send Alerts to System Log' will be auto-enabled.
---

# Phase 4 - Download Rules
Go to

```
Services
Suricata
Global Settings
```
Install ETOpen Emerging Threats rules ✔ Enable
Install Snort GPLv2 Community rules ✔ Enable

SAVE

Go to

```
Services
Suricata
Updates
```

Enable one or more rule sources:

### Emerging Threats Open

Free

```
✔ Enable ET Open
```

---

### Snort Subscriber Rules

Only if you have an Oinkcode.

---

Click

```
Update Rules
```

---

# Phase 5 - Enable Rules

Go to

```
Services
Suricata
Interfaces [select the interface]
[Interface Name] Categories
```

Initially enable:

```
emerging-malware

emerging-exploit

emerging-web_server

emerging-dns

emerging-shellcode

emerging-scan

emerging-trojan

emerging-icmp

emerging-policy
```

Avoid enabling every category immediately, as it can create a lot of noise.

---

# Phase 6 - Test

From Kali

```
ping 10.10.30.10

nmap -A <ipaddress>

nikto

hydra

msfconsole
```

You should start seeing alerts in

```
Services

Suricata

Alerts
```

---


# Phase 7 - Connect to logCollectorVM
Although Suricata allows us to assign multiple Remote Log Servers (three on pfSense), sending the logs directly to the splunk and wazuh servers caused issues where only one of the servers received the logs. In order to bypass this issue, we are sending the logs to a seperate VM which is dedicated for the purpose of log collection. 

### Step 1: Provision the VM in Proxmox
Go to your Proxmox dashboard and create a new virtual machine with the following specs:
OS: Ubuntu Server (24.04 LTS or newer)
CPU: 1 or 2 cores
RAM: 1 GB to 2 GB
Storage: 20 GB to 30 GB (Thin provisioned)
Network: Attach it to your internal lab network bridge (vmbr1 or whichever bridge your other VMs are using).
Complete the Ubuntu installation, set your username, and make sure to check the box to Install OpenSSH Server so you can manage it cleanly.

### Step 2: Configure a Static IP on the Log VM
Assign a static ip to the VM from pfsense DHCP server for MGMT interface.

### Step 3: Configure `rsyslog` to Accept pfSense Logs & Forward to Splunk

Ubuntu handles syslog data natively using `rsyslog`. We just need to turn on its listening ears and tell it where to copy the data.

1. Open the main config file:
    
```
    sudo nano /etc/rsyslog.conf
```
    
2. Scroll down and find the **UDP syslog reception** lines. Delete the `#` comment symbols in front of them so they look exactly like this:
    
```
    # provides UDP syslog reception
    module(load="imudp")
    input(type="imudp" port="514")
```
    
3. Now scroll to the very bottom of the file and paste this custom rule block. This creates a dedicated file for pfSense logs and mirrors a copy right over to Splunk via UDP:
    
```
    # Create a local file rule for pfSense logs
    if $fromhost-ip == '10.10.1.1' then /var/log/pfsense.log
    
    # Forward a live copy of ALL incoming logs to Splunk
    *.* @10.10.1.101:514
```
    
4. Save and exit, then restart `rsyslog` to boot up the listener:Bash
    
```
sudo systemctl restart rsyslog
```
    

### Step 4: Point pfSense to the New Forwarder VM

Now we update the firewall's logging settings to point to our new middleman.

1. Log into your **pfSense Web GUI**.
2. Navigate to **Status > System Logs > Settings**.
3. Scroll to the bottom to **Remote log servers**.
4. In **Box 1**, change the IP to your new collector: `10.10.1.106:514` (this is the IP for the logCollectorVM)
5. Make sure Box 2 and Box 3 are completely **blank** (preventing that validation error).
6. Remote Syslog Contents ✔ Everything
7. Click **Save** at the bottom.
8. Clear out the pfSense cache by running the reload command under **Diagnostics > Command Prompt**:
    
```
killall -HUP syslogd
```
    
### Step 5: Verify the Splunk Pipeline is Alive

At this stage, pfSense is sending logs to `10.10.1.106`, and the Log VM is writing them down locally while simultaneously throwing them over to Splunk.

Let's make sure Splunk is still receiving data under this new model:

1. Run a test attack from your **Kali VM**: `curl http://testmyids.com`
2. Look at your **Log VM terminal** to check if the file is writing data:
    
```
sudo tail -n 20 /var/log/pfsense.log
```
If pfsense.log file has logs being populated in it means that suricata is sending logs to the log collector vm.

### Step 6: Ensure Splunk is listening on port 514
Splunk does not listen on port 514 by default. We need to make sure you have explicitly created a network input inside Splunk to catch these packets.

Open your Splunk Web UI (192.168.1.203:8000).

Navigate to Settings > Data Inputs (in the top right menu bar).

Look for UDP in the list and click + Add New next to it.

Configure the input settings exactly like this:

Port: 514

Source name override: pfsense (optional, helps keep things organized)

Click Next.

On the Input Settings page:

Source Type: Click Select and type syslog (or choose pfsense if you have the pfSense Add-on installed).

App Context: Search & Reporting

Index: main (or a dedicated index if you made one).

Click Review and then Submit.

![Alt Text](/images/Splunk-syslog514.png)

*NOTE: If Splunk doesn't allow the use of port 514 substitute it with a different port number such as 5140 or 1514 [The port needs to be unused]. And update the rsyslog.conf file on the log collector vm with the same port number*

Check your **Splunk dashboard**. You should see the fresh syslog entries indexing cleanly!

QUERY:
```
source="pfsense" index="main" sourcetype="syslog"
```

### Install the Wazuh Agent on the Log Collector VM to seamlessly bridge the gap into the XDR dashboard
The collector VM is successfully intercepting the pfSense logs, stamping its own telemetry alongside it, and pushing it straight into Splunk.

Now for the final phase: installing the **Wazuh Agent** on this log collector VM so Wazuh can natively read `/var/log/pfsense.log` from the disk and send it over to your XDR manager.

### Step 1: Install the Wazuh Agent on the Log Collector VM

GET THE COMMAND TO ADD UBUNTU ENDPOINT TO WAZUH FROM WEB GUI

### Step 2: Point the Agent to the pfSense Log File

Now we need to tell this newly installed agent to actively monitor our flat-file syslog repository.

1. Open the local agent configuration file:Bash
    
```
sudo nano /var/ossec/etc/ossec.conf
```
    
2. Scroll down until you see the `<ossec_config>` section blocks. Look for where other `<localfile>` configurations are declared, and paste this block directly among them:XML

ADD THE BELOW LINES DO NOT REPLACE THE EXISTING SYSLOG FILENAME
    
```
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/pfsense.log</location>
  </localfile>
```
    
3. Save the file and exit (`Ctrl+O`, `Enter`, `Ctrl+X`).
4. Check file permissions

Verify the agent can read the log:

```
ls -l /var/log/pfsense.log
```

Typical output:

```
-rw-r----- 1 syslog adm 897059 Jul 15 08:34 /var/log/pfsense.log
```

Check what users are in the “adm” group
```
getent group adm
```
Output:
```
adm:x:4:syslog,ash,wazuh
```
If wazuh is not in adm group add it to the group
```
sudo usermod -aG adm wazuh
```
Check the Wazuh user:

```
ps aux |grep wazuh
```

If necessary:

```
sudo chmod 644 /var/log/pfsense.log
```

Restart the agent:

```
sudo systemctl restart wazuh-agent
```
### Step 3: Boot Up the Agent

Enable the agent service daemon so it automatically starts up whenever the VM boots, and kick the process into action:


```
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

### Step 4: The Ultimate Test

Now that the entire pipeline is linked, let's verify if everything lands on your dashboards.

1. Head over to your **Kali Linux machine** and trigger the signature exploit string:Bash
```
curl http://testmyids.com
```
    
2. Check your **Wazuh Dashboard** interface under the "Security Events" or "Discover" tab.

Because Wazuh's manager includes native out-of-the-box decoding decoders for Suricata IDS telemetry, it will automatically extract the signature payload from the agent's file stream and drop a beautifully formatted alert on your threat map dashboard.


### CHECK IF pfsense.log is being populated
```text
sudo grep -i suricata /var/log/pfsense.log | tail -20
```
**NOTE**:If logs are being populated ont the log collector VM but not showing up on the wazuh dashboard the issue is on the wazuh server. Pivot to troubleshooting wazuh server.!!!!!!!!!!!!!!!!!!!!!!!!!!!!