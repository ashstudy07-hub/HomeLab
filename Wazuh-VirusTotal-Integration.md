## VirusTotal Integration

This is a followup from the Wazuh-setup-config.md.
This lab combines the File Integrity Monitor (FIM) feature of Wazuh and combines it with Virustotal to provide an effective way of monitoring and inspecting monitored files for malicious content.

We will be using a **FREE** public API from virustotal in this lab. 
The free API has a limited usage quota>

|Access level|Limited|
|---|---|
|Usage|	Must not be used in business workflows, commercial products or services.|
|Request rate|	4 lookups / min|
|Daily quota|	500 lookups / day|
|Monthly quota|	15.5 K lookups / month|

### Use Case
## Integrate VirusTotal to the Wazuh Server
Below is an example of settings you must add to the /var/ossec/etc/ossec.conf file on the Wazuh server:

```text
<integration>
  <name>virustotal</name>
  <api_key>API_KEY</api_key> <!-- Replace with your VirusTotal API key -->
  <group>syscheck</group>
  <alert_format>json</alert_format>
</integration>
```
Add it at the end of the ossec.conf file before the **</ossec_config>** line.

## Using FIM to monitor a directory
For this use case, we show how to monitor the folder /home/user/Downloads on Linux endpoints.

Add the following to the <syscheck> section of the configuration file. You can configure these options in the Wazuh server and the Wazuh agent /var/ossec/etc/ossec.conf file. You can also configure this capability remotely using the centralized configuration options provided by the agent.conf file. The list of all FIM configuration options is available in the [syscheck](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/syscheck.html) section of the documentation. In our example, we configured the options below on the Wazuh server using the **agent.conf** file.
In the previous lab we created groups in wazuh and added the agents to the group matching their OS. By doing so we can now monitor the Downloads folder on all Linux desktop by applying the configuration to the agent.conf file of that group. 

Select the Hamburger icon on the top left of the Web GUI. > Click on **Agents management** > Select **Groups** > Select the Edit group configuration (pencil icon) on the desired group. > Add the below configuration.

```text
  <agent_config os="Linux">
    <!-- Shared agent configuration here -->
    <syscheck>
      <directories realtime="yes" check_all="yes">/home/*/Downloads</directories>
    </syscheck>
  </agent_config>
```

After applying the configuration, you must restart the Wazuh manager:

```text
systemctl restart wazuh-manager
```

After restarting, FIM applies the new configuration and monitors the folder you specify in near real time.

## Test the configuration
Now, you can download a malicious file on the endpoint in the monitored folder.

```text
sudo curl -Lo /media/user/software/suspicious-file.exe https://secure.eicar.org/eicar.com
```
