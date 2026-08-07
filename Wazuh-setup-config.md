<p align="center">
  <img src="images/wazuh.png" width="400">
</p>


This is a quickstart installation quide for Wazuh XDR/SIEM.
The soultion is composed of a single universal agent and three central components: the Wazuh server, the Wazuh indexer, and the Wazuh dashboard.
For more information, check the official [Getting Started](https://documentation.wazuh.com/current/getting-started/index.html) documentations.
## Hardware Requirements

| Agents | CPU | RAM | Storage (90 days) |
|--------|-----|-----|-------------------|
| 1-25 | 4 vCPU | 8 GiB | 50 GB |
| 25-50 | 8 vCPU | 8 GiB | 100 GB |
| 50-100 | 8 vCPU | 8 GiB | 200 GB |

## Important Directories and files

| Directories | Files |
|-------------|-------|
|/etc/wazuh-indexer/|opensearch.yml & certificates|
|/etc/filebeat/| filebeat.yml, wazuh-template.json and certificates|
|/etc/wazuh-dashboard|opensearch_dashboards.yml and certificates|
|/usr/share/wazuh-dashboard/data/wazuh/config/|wazuh.yml|
|/var/ossec/logs/|archives/archives.json, alerts.json|
|/var/ossec/logs/|ossec.log|
|/var/ossec/etc/|**ossec.conf**|
|/var/ossec/active-response/bin|Active response commands scripts|
|/var/ossec/rules|Rules files|
|/var/ossec/etc/rules/|**local_rules.xml**|
|/usr/share/wazuh-indexer/plugins/opensearch-security/tools/|wazuh-passwords-tool.sh|

## Ubuntu Server setup
For the purpose of this lab we will be using [Ubuntu 26.04 LTS](https://ubuntu.com/download/desktop).

Beffore we begin the wazuh installation process, it is essential to ensure that your APT (Advanced Package Tool) repository is up-to-date.
This step ensures that you have access to the latest package information and versions.

Open your terminal and run the following command:
```text
sudo apt update && sudo apt upgrade -y
```
## Installing Wazuh 
1. Download and execute the Wazuh installation assistant script with the following commands. 
This script simplifies the installation process, guiding you through the setup of Wazuh.

```text
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```
Once the assistant finishes the installation, the output shows the access credentials and a installation successful message.


```text
INFO: --- Summary ---
INFO: You can access the web interface https://<WAZUH_DASHBOARD_IP_ADDRESS>
      User: admin
      Password: <ADMIN_PASSWORD>

INFO: Installation finished.
```
2. Access the Wazuh web interface with ```https://<WAZUH_DASHBOARD_IP_ADDRESS>``` and your credentials:

Username: admin

Password: <ADMIN_PASSWORD>

Note: You can find the passwords for all the Wazuh indexer and Wazuh API users in the wazuh-passwords.txt file inside wazuh-install-files.tar. To print them, run the following command:

```text 
sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt
```

## Change Admin Password
To change the Wazuh admin password, use the wazuh-passwords-tool.sh script via the command line, as the web interface may restrict direct changes for security reasons. 

First, download the tool on the Wazuh Manager node using:
```text
curl -so wazuh-passwords-tool.sh https://packages.wazuh.com/4.14/wazuh-passwords-tool.sh
```
To check if it is downloaded, run the following command in your terminal:
```text
ls -l /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-t
```

Then, execute the script with the -u flag for the user and -p for the new password:
```text
sudo bash wazuh-passwords-tool.sh -u admin -p <NEW_PASSWORD>
```

Ensure the new password meets security requirements: 8–64 characters, including uppercase, lowercase, numbers, and symbols like .*+?-.  After changing the password, restart the Wazuh Dashboard service to apply the changes:
```text
systemctl restart wazuh-dashboard
```

## Create new Admin Account and Read-only user
Creating new users can be done using the Web GUI.
Follow the official [instructions](https://documentation.wazuh.com/current/user-manual/user-administration/rbac.html#creating-and-setting-a-wazuh-admin-user) provided by Wazuh to complete this step.


## Change Wazuh Dashboard default Timeout
By default the wazuh dashboard timesout after 15 minutes (900,000 ms) of inactivity.
You can extend the **Wazuh** dashboard session timeout by modifying the configuration file on the server.  This prevents automatic logouts due to inactivity by increasing the time before the session expires.

To implement this, edit the `/etc/wazuh-dashboard/opensearch_dashboards.yml` file and add or modify the following lines, setting the values in **milliseconds**:

```
opensearch_security.session.ttl: 43200000
opensearch_security.cookie.ttl: 43200000
opensearch_security.session.keepalive: true
```

- **43200000** equals 12 hours; adjust this number as needed (e.g., 86400000 for 24 hours).
- After saving the file, restart the dashboard service using `sudo systemctl restart wazuh-dashboard` for the changes to take effect.


## Deploy Wazuh Agent
You can deploy a new agent following the instructions in the Wazuh dashboard. Go to Agents management > Summary, and click on Deploy new agent.

![Alt Text](/images/Wazuh-agent-dashboard1.png)

On the **Deploy new agent**:
1. Select the Operating System type of the end host.
2. Enter the IP address of the Wazuh Server.
3. You can manually assign an agent name or leave it blank to use the default device name of the end host.
4. Wazuh will generate a command that is specific to the OS of the end host to download and install the agent. Run the command on the end host.
5. Once the command runs successfully start the Wazuh agent.

![Alt Text](/images/Wazuh-agent-deployment.png)

**NOTE**: Anytime you make a change to the wazuh agent or server, you need to restart the service for the changes to take effect.

Wazuh Agent and Manager use Ports 1514 and 1515 to communicate. Allow traffic on these ports on the firewall!

![Alt Text](/images/Wazuh-firewall-port-alias.png)
![Alt Text](/images/Wazuh-firewall-rule.png)


| Index | Purpose |
| --- | --- |
| wazuh-alerts-* | Security alerts and detections |
| wazuh-monitoring-* | Wazuh component health |
| wazuh-statistics-* | Manager statistics |
| wazuh-states-vulnerabilities-* | Vulnerability scans |
| wazuh-states-inventory-* | Asset inventory |

## Assign End hosts to Groups based on their OS
Assigning end hosts that have wazuh agents on then to groups will make managing them easier.
1. Select the Hamburger icon on the top left of the Web GUI.
2. Click on **Agents management**
3. Select **Groups**

![Alt Text](/images/agent-groups.png)

4. Click the +Add new group icon and create group (e.g., Windows, Linux-desktop, Linux-server)
5. To add agents to the group, select the group you want to add the agents to and click the **Manage agents** button.
6. Select the agents from the available agents box and add them to the current group.
7. Apply changes.
