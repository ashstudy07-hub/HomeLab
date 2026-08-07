# SSH to a different LAN on proxmox
Scenario 

Management PC: 192.168.1.202

Proxmox: 192.168.1.210

pfsense: 192.168.1.215, 10.20.10.1

jumpboxVM on pfsense: 10.20.10.200

The purpose of this Lab is to SSH into the LAN subnet [10.20.10.x] inside proxmox from a management PC connected to the home network [not a VM on Proxmox]

**The Conversation Flow**
1. **The Request:** `192.168.1.202` $\rightarrow$ `192.168.1.215` (WAN) $\rightarrow$ `10.20.10.105` `[SYN]`
2. **The Reply:** `10.20.10.105` $\rightarrow$ `10.20.10.1` (MGMT/LAN) $\rightarrow$ `192.168.1.202` `[SYN/ACK]`

Management PC
192.168.1.202
|
|  Route: 10.20.0.0/16 via 192.168.1.215
|
pfSense WAN
192.168.1.215
|
pfSense
|
10.20.10.0/24
|
Jumpbox
10.20.10.200

#### Step 1: Create a static route on the Management PC

The static route directs the 10.20.10.x traffic to the pfSense Gateway [192.168.1.215] on the same subnet as the Management PC 

Note: Management PC is a Windows 

Check Route
```
route print
```
Add new Route [requires elevated privilege] 
```
route -p 10.20.0.0 mask 255.255.0.0 192.168.1.215
```
Note: 10.20.0.0 is being used instead of 10.20.10.0 because in this use case all 10.20.x.x will go through the 192.168.1.215 firewall in future

#### Step 2: Add the static Route on the Bridge Router

Advanced> Network > Static Routing

![Alt Text](/images/StaticRoute.png)

#### Step 3: Create Portforwarding rule on pfSense Firewall

ON pfSense firewall

Source IP: IP address of Mangement PC

Destination: WAN address [address of pfsense WAN]

Destination port: 2222 [choose an unused port number]

Redirect target IP: IP address of the Jumpbox

Redirect target port: SSH

![Alt Text](/images/portforwarding-sshrule.png) 

The port forwarding rule will automatically generate a firewall rule to allow SSH traffic from the Management PC to the Jumpbox.

#### Step 4: Ensure the Jumpbox vm on proxmox’s Internal LAN has SSH enabled

Check status
```
systemctl status ssh
```
Start and enable ssh if it is stopped and disabled 
```
systemctl start ssh

systemctl enable ssh
```
This allows you to ssh into the Internal Subnet [10.20.10.x] from the “WAN” subnet which is the IP assigned by the home subnet [192.168.1.x]

#### Step 5: Use the SSH `ProxyJump` Command

Once that single rule is active, you don't need any other port forwards for any other VMs. You can tell your Management PC to automatically "hop" through the Jumpbox to reach any destination in the internal network using a single command in PowerShell:

PowerShell

```
ssh -J jumpbox_user@192.168.1.215:2222 user@10.20.10.105
```

(Where `-J` stands for ProxyJump. Your PC connects securely to the pfSense WAN IP on port 2222, tunnels through the Jumpbox, and instantly drops you onto the Wazuh server or any other internal asset).

#### Option A: The One-Liner Command (ProxyJump)

To connect straight to another target machine (for example, your Wazuh server at `10.20.10.105`) by tunneling right through Kali, use the `-J` flag:

PowerShell

```
ssh -J jumpbox_user@192.168.1.215:2222 target_user@10.20.10.105
```

*Replace `jumpbox_user` with your Jumpbox username (e.g., `kali`) and `target_user` with the username of the destination VM.*

#### Option B: SSH Config File (The Cleanest Way)

If you don't want to type out a long command every time, you can configure your Management PC to handle this automatically.

1. On your Windows Management PC, open or create the file `C:\Users\YourUsername\.ssh\config` in a text editor.
2. Paste the following configuration block:

Plaintext

```
# Define the Kali Jumpbox shortcut
Host kali-jump
    HostName 192.168.1.215
    Port 2222
    User kali

# Automatically route any 10.20.x.x connection through the Jumpbox
Host 10.20.*.*
    ProxyJump kali-jump
```

1. Save the file. Now, whenever you want to connect to *any* machine in that subnet, your PC will transparently tunnel through the Kali VM. You can simply type:

PowerShell

```
ssh target_user@10.20.10.105
```

Because all of the routing is handled encapsulated inside the symmetric NAT connection to port `2222`, you will completely bypass the asymmetric state tracking issues, and you won't need to build any individual rules for future lab VMs!

### The Step-by-Step Authentication Flow

1. **Step 1: The Jumpbox Handshake**
Your Management PC sees you want to hit a `10.10.1.*` address and looks at your config. It opens a secure SSH connection to `192.168.1.215` on port `2222`. pfSense simply acts as a traffic router here—it silently passes the encrypted packets straight through to your Kali VM.
    - **The First Prompt:** The first password or SSH key you provide goes directly to the **Kali Linux VM** to log in as the user `kali`.
2. **Step 2: The Tunnel Establishment**
Once Kali authenticates you, it doesn't drop you into a terminal command line. Instead, it opens a secure cryptographic "tunnel" inside that existing connection, stretching from your Management PC, through Kali, directly to the destination VM's port 22.
3. **Step 3: The Target Handshake**
Your Management PC now speaks directly to the end machine through that tunnel.
    - **The Second Prompt:** The second password or SSH key you enter goes directly to the **Destination VM** (e.g., logging in as `user` or `root` on your Wazuh or Splunk server).

### Pro-Tip: How to Get Rid of Password Prompts Entirely

Entering two passwords every time you want to access a VM can get old fast. You can use **SSH Keys** to make this completely seamless and instant:

1. **Generate an SSH Key on your Management PC** (if you don't have one):PowerShell
    
    ```
    ssh-keygen -t ed25519
    ```
    
2. **Copy that Key to your Kali Jumpbox** (authenticate with its password one last time):PowerShell
    
    ```
    ssh-copy-id -p 2222 jumpbox_user@192.168.1.203
    ```
    
3. **Copy that Key to your Destination VMs** (tunneling through Kali):PowerShell
    
    ```
    ssh-copy-id -o ProxyJump=kali-jump user@10.10.1.105
    ```
    

Once your public key is added to both systems, typing `ssh user@10.10.1.105` will instantly log you all the way into the final VM in less than a second without prompting you for a single password!


### For Security


Enforcing key-based authentication means your Management PC will use a pair of cryptographic keys to log in, and you will disable password authentication on the Kali Jumpbox entirely.

Here is how to set it up step-by-step from your Windows Management PC and your Kali VM.

### Step 1: Generate an SSH Key Pair on your Management PC

If you don't already have an SSH key on your Windows machine, open a standard **PowerShell** window and run:

PowerShell

```
ssh-keygen -t ed25519
```

- **What to do:** Press **Enter** to accept the default file location (`C:\Users\Username\.ssh\id_ed25519`).
- **Optional:** Enter a passphrase if you want an extra layer of local security, or leave it blank by pressing Enter twice.

This creates two files in your hidden `.ssh` directory:

1. `id_ed25519` (Your **Private Key** — keep this safe on your PC and never share it).
2. `id_ed25519.pub` (Your **Public Key** — this is the lock you will upload to Kali).

### Step 2: Copy your Public Key to the Kali Jumpbox

Because your Kali Jumpbox is hiding behind the pfSense NAT on port `2222`, you need to tell Windows to copy the key explicitly across that custom port.

In your Windows PowerShell window, run the following command (replace `jumpbox_user` with your actual Kali username if different):

PowerShell

```
cat C:\Users\Username\.ssh\id_ed25519.pub | ssh jumpbox_user@192.168.1.215 -p 2222 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

- *Note: Replace the "Username" to your own username. Type your Kali password one last time when prompted to allow the file upload.*

### Step 3: Test Key-Based Login

Before changing any settings on Kali, verify that the key exchange worked. Run your standard shortcut command from PowerShell:

PowerShell

```
ssh target_user@10.10.1.105
```

*(Or your fallback manual command: `ssh jumpbox_user@192.168.1.203 -p 2222`)*

**If it works correctly:** The Kali Jumpbox step should log you straight through *without* prompting you for your Kali password (it will only ask for the end destination password at `10.10.1.105`).

### Step 4: Disable Password Authentication on Kali

Now that your key is verified to be working, you can completely pull the plug on passwords so nobody can try to brute-force your jumpbox.

1. While logged into your **Kali VM terminal**, open the SSH daemon configuration file:Bash
    
    ```
    sudo nano /etc/ssh/sshd_config
    ```
    
2. Scroll down or use `Ctrl + W` to search for the following lines. Change them to match this configuration:Plaintext
    
    ```
    PubkeyAuthentication yes
    PasswordAuthentication no
    PermitEmptyPasswords no
    ```
    
    *(Make sure there isn't a `#` symbol in front of these lines. If there is, delete the `#` to uncomment them).*
    
3. Save and exit Nano by pressing `Ctrl + O`, then `Enter`, then `Ctrl + X`.
4. Restart the SSH service to enforce the new rule:Bash
    
    ```
    sudo systemctl restart ssh
    ```
    

### The Final Result

Open a brand new PowerShell window on your Management PC and test your configuration.

- Your PC will use the seamless `id_ed25519` key to transparently clear the Kali hurdle.
- If anyone else on your home network tries to attack your pfSense port `2222` using a password cracker, Kali will instantly drop their connection right at the handshake level with a `Permission denied (publickey)` error. Your jumpbox pivot is now fully hardened!



### Port Forward for Splunk VM
*Note: The default port for Splunk web Gui is 8000*

Source IP: [Address or Alias] IP address of Mangement PC

Destination: WAN address [address of pfsense WAN]

Destination port: 8000

Redirect target IP: IP address of the Jumpbox

Redirect target port: Other
custom: 8000

![Alt Text](/images/portforwarding-splunk.png)

