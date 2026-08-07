## Splunk Setup and Configuration

## Hardware Requirements

|CPU|MEMORY|Storage|
|---|------|-------|
|4|8 GB|160GB|

## Important Directories and files

Default Install location  

cd /opt/splunk

| Directories | Files |
|-------------|-------|
|/opt/splunk/bin/|splunk|

## Ubuntu Server setup
For the purpose of this HomeLab Environment we will be using [Ubuntu 26.04 LTS](https://ubuntu.com/download/desktop).

Beffore we begin the wazuh installation process, it is essential to ensure that your APT (Advanced Package Tool) repository is up-to-date.
This step ensures that you have access to the latest package information and versions.

Open your terminal and run the following command:
```text
sudo apt update && sudo apt upgrade -y
```
## Installing Splunk

Get the [SPLUNK ENTERPRISE](https://www.splunk.com/en_us/download/splunk-enterprise.html) for ubuntu


You need a Splunk account to get access to the download.

Once you have created an account and logged in, Copy the wget link for .deb Linux as we age using a ubuntu VM.
Run the command that looks like the one below on the Splunk VM.
```text
wget -O Splunk-10.4.0-f798d4d49089-linux-amd64.deb "https://download.splunk.com/products/splunk/releases/10.4.0/linux/splunk-10.4.0-f798d4d49089-linux-amd64.deb"
```

Check if Splunk is downloaded
```text
ls 
```

Install Splunk
```text
sudo dpkg -i splunk-10.4.0-f798d4d49089-linux-amd64.deb
```

Look at the binaries and python scripts [we are interested in splunk file]
```text
cd /opt/splunk/bin
```
Start the splunk binary
```text
./splunk start
```

Running Splunk Enterprise as root is deprecated and will be removed in a future release. For details, see the Release Notes. To run as root, use the --run-as-root option.
if this happens
```text
./splunk start --run-as-root
```
scroll to the end and agree to the license

You wont be able to access the login screen if port **8000** is not enabled in the firewall

ufw allow 8000

ufw status

if you have a firewall on the cloud provider make a rule there too 

## Start splunk on Boot

Step 1: Fix Permissions on the var Directory
Ensure the splunk account recursively owns the entire Splunk directory:

sudo chown -R splunk:splunk /opt/

Step 2: Run the Correct Command
Run the command using boot-start:
```text
sudo /opt/splunk/bin/splunk enable boot-start -user splunk -systemd-managed 1
```
Step 3: Check Status
systemctl status Splunkd.service


## Troubleshoot Installation
At the end of installation instead of receiving the url to access the web GUI, if you get the following warning message:
```
Warning: cannot create "/opt/splunk/etc/licenses/download-trial"
```
Stop any running Splunk instances and fix permissions:
```
sudo pkill -f splunk
sudo chown -R splunk:splunk /opt/splunk
```
Start Splunk as the splunk user to accept the license and initialize the instance:
```
sudo -i -u splunk /opt/splunk/bin/splunk start --accept-license
```

This will ask you to create an admin account to access Splunk GUI.

When attempting to start Splunk on boot if you get the following Error:
```
"splunk is currently running, please stop it before running enable/disable boot-start"
```
1. Check systemctl status for running processes
```
systemctl list-units --type=service --state=running
OR
systemctl list-units --type=service --state=running | grep -i '[s]plunk'
```
If you see a splunk process running stop the process using
```
sudo systemctl stop <service-name>
```

However, if you do not see Splunk service listed when running the above command, you have to manually stop the Splunk service.
```
sudo /opt/splunk/bin/splunk stop   
```

Enable Boot-Start
```
sudo /opt/splunk/bin/splunk enable boot-start -user splunk -systemd-managed 1   
```

Reload and Verify
```
sudo systemctl daemon-reload
systemctl status Splunkd   
```

If the status shows active (running), you can now manage Splunk entirely via systemctl (e.g., sudo systemctl start Splunkd). If it shows inactive, start it manually with sudo systemctl start Splunkd.


## Configure Splunk
Change time to GMT

Administrator> Preference> Time zone > GMT> Apply

Add app for windows [this get the “user” field for the device when searching in splunk]

Under Apps> Find more apps> search for “Windows”> **Splunk Add-on for Microsoft Windows > Install
login using the splunk username and pass that you used to download Splunk [not the one used to login to splunk]**

Create a new index

Settings > [under DATA] indexes> new index> name - mydfir-ad> save

Configure Receiving port

Settings> [Under DATA] forwarding and receiving> [under Receive data] Configure receiving > New Receiving Port> Listen on this port * → 9997 
this is the default port

You have to modify firewall to allow incoming on port 9997

on ubuntu server with splunk server installed

ufw allow 9997


## Install Universal Forwarder for client device
1. Download UNIVERSAL FORWARDER for the client devices

2. Copy the forwarder to the client device and run it

3. Keep it as An on-premises Splunk enterprise instance

4. Next

5. username: mydfir
generate random password

6. There is no deployment server [leave empty]
Next

7. Receiving indexer
Hostname or IP
private IP of the ubuntu server with Splunk server running
and port number specified in the last step (9997)
8. Install

Admin and password of “DOMAIN CONTROLLER” not local admin (if it is domain joined endpoint)


## Endpoint agent configuration

Copy inputs.conf from → C:\Program Files\SplunkUniversalForwarder\etc\system\default

TO

C:\Program Files\SplunkUniversalForwarder\etc\system\local

Edit inputs.conf using notepad with administrative privilege

Add these lines at the end of the inputs.conf file [index name is the name that was created earlier]

[WinEventLog://Security]
index = mydfir-ad
disable = false

Save the file then open “Services” as Administrator of DC to RESTART Splunk Forwarder

double click Splunk Forwarder > logon> Select Local System account> apply > ok 

Then click restart on the left
if you get error message click ok 
and start the splunk forwarder

On the splunk web dashboard 

Search 
index="mydfir-ad”

If the forwarder setup was done correctly logs will start to populate

Repeat the process for all clients you want the logs from

## Install Universal Forwarder on a Linux client device
1. Download UNIVERSAL FORWARDER installation Package for Linux. Copy the wget link and run it.
    The command looks like:
```
  wget -O splunkforwarder-10.4.2-33c3bf42cd73-linux-amd64.deb "https://download.splunk.com/products/universalforwarder/releases/10.4.2/linux/splunkforwarder-10.4.2-33c3bf42cd73-linux-amd64.deb"
```

2. Install the Downloaded Package
Run dpkg to install the .deb file:

```
sudo dpkg -i splunkforwarder-10.4.2-33c3bf42cd73-linux-amd64.deb 
```
(This installs the forwarder files to /opt/splunkforwarder.)

3. Start Splunk and Create Admin Credentials
Start the service, accept the license agreement, and enter an admin username and password when prompted:

```
sudo /opt/splunkforwarder/bin/splunk start --accept-license
```
You will be prompted to create an admin account.
4. Enable Boot-Start (Systemd)
Check if Splunk Forwarder is running.

```
sudo systemctl status SplunkForwarder.service
```
If it is running you need to STOP the service 

```
sudo systemctl stop SplunkForwarder.service
```
Configure the service to start automatically whenever the system reboots:

```
sudo /opt/splunkforwarder/bin/splunk enable boot-start
```
5. Connect to Your Splunk Indexer
Configure the forwarder to send data to your central Splunk Indexer (default receiving port is 9997):

```
sudo /opt/splunkforwarder/bin/splunk add forward-server <INDEXER_IP_OR_HOSTNAME>:9997
```
6. (Optional) Connect to a Deployment Server
If you manage forwarder configurations centrally using a Splunk Deployment Server:

```
sudo /opt/splunkforwarder/bin/splunk set deploy-poll <DEPLOYMENT_SERVER_IP>:8089
```
7. Verify Connection Status
Confirm that the forwarder is connected to your receiving target:

```
sudo /opt/splunkforwarder/bin/splunk list forward-server
```

## Configure the data that the forwarder collects and sends [On linux client]

1. Configure Data InputsTo collect logs (such as system logs, web server logs, or custom app logs), add them to /opt/splunkforwarder/etc/system/local/inputs.conf or use the CLI.Example using the CLI (monitor a log file):
```
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/syslog -index main -sourcetype syslog
```
Example editing inputs.conf directly:
Open the local configuration file using sudo:

```
sudo nano /opt/splunkforwarder/etc/system/local/inputs.conf
```
Add your monitoring stanzas. For example:
```
[default]
host = your-hostname

[monitor:///var/log/syslog]
disabled = 0
index = main
sourcetype = syslog
```
2. Restart the Forwarder
If you manually edit configuration files like inputs.conf, restart Splunk for changes to take effect:
```
sudo /opt/splunkforwarder/bin/splunk restart
```
3. Verify Data in Splunk WebLog into your central Splunk Web console on the indexer and run a search to verify data arrival:
```
host="<hostname-of-this-arm64-machine>"
```
## Troubleshoot syslog not populating
Test if syslog logs are being generated
```
logger -t splunk_test "SPLUNK TEST LOG: Verification event sent at $(date)"
```

Kali Linux uses systemd-journald for logging by default and does not ship with a traditional syslog daemon (like rsyslog) or a /var/log/syslog file enabled out of the box.

When you ran logger, the test message was sent successfully to journald, but /var/log/syslog doesn't exist on your filesystem.

Here are two ways to resolve this depending on how you prefer to handle logs:

Option 1: Monitor journald directly in Splunk (Recommended for Kali)
Since Kali uses systemd, configuring Splunk to pull directly from journald avoids installing unnecessary packages.

Open /opt/splunkforwarder/etc/system/local/inputs.conf:

```
sudo nano /opt/splunkforwarder/etc/system/local/inputs.conf
```

Add a journald input stanza:

```
[journald://systemd]
disabled = 0
index = homelab
sourcetype = journald
```

Restart the forwarder:

```
sudo /opt/splunkforwarder/bin/splunk restart
```

Verify your test log using journalctl locally:

```
sudo journalctl -t splunk_test
```

Option 2: Install rsyslog to create /var/log/syslog
If you specifically want traditional log files in /var/log:

Install and start rsyslog:

```
sudo apt update && sudo apt install rsyslog -y
sudo systemctl enable --now rsyslog
```

Re-run your test command:

```
logger -t splunk_test "SPLUNK TEST LOG: Verification event sent at $(date)"
```
Confirm /var/log/syslog now exists and contains the event:

```
sudo tail -n 5 /var/log/syslog | grep "SPLUNK TEST LOG"
```

## Sample query for this index
```text
index="mydfir-ad"
| stats count by _time, ComputerName, Source_Network_Address,user, Logon_Type
```





