# API Wizard Breach (Forensics)

## Task 1: Preparation

### Step 1.1 SSH into the Machine

SSH into the box with the credentials provided.

```bash
ssh <username>@<IP_address>
# Replace <username> and <IP_address> with the values given

```

---

## Task 2: Initial Access

### Question 1: Which programming language is the web application written in?

Navigate through the directories to find the application’s code.

```bash
cd /home/support/api_service
ls -la

```

Answer: **Python**

### Question 2: What is the IP address that attacked the web server?

The web server uses NGINX, so we can check its logs for suspicious activity.

```bash
cd /var/log/nginx
tail -n 10 access.log.1

```

Look for commands like `whoami`, `pwd`, or `id`, which indicate enumeration attempts by an attacker.

Answer: **149.34.244.142**

### Question 3: Which vulnerability was found and exploited in the API service?

Inspect the `api.py` source code for vulnerabilities, particularly around command handling.

```bash
cat /home/support/api_service/api.py

```

Answer: **OS command injection**

### Question 4: Which file contained the credentials used to privilege escalate to root?

Examine the configuration file for any stored credentials.

```bash
cat /home/dev/apiservice/src/config.py

```

Answer: **/home/dev/apiservice/src/config.py**

### Question 5: What file did the hacker drop and execute to persist on the server?

Check the bash history to uncover any evidence of files dropped.

```bash
sudo su
cat /root/.bash_history

```

Answer: **/tmp/rooter2**

### Question 6: Which service was used to host the “rooter2” malware?

In `.bash_history`, look for commands involving file uploads or downloads.

Answer: [**transfer.sh**](http://transfer.sh/)

---

## Task 3: Further Actions

### Question 7: Which two system files were infected to achieve cron persistence?

Check `crontab` and environment files for unauthorized entries.

```bash
cat /etc/crontab
cat /etc/environment

```

Answer: **/etc/crontab, /etc/environment**

### Question 8: What is the C2 server IP address of the malicious actor?

Locate the IP address associated with the `SYSTEMUPDATE` variable in `/etc/environment` or other files identified in bash history.

Answer: **5.230.66.147**

### Question 9: What port is the backdoored bind bash shell listening on?

Use `ps` to check running processes for a netcat listener.

```bash
ps -aux | grep nc

```

Answer: **3578**

### Question 10: How does the bind shell persist across reboots?

Locate the systemd service created by the attacker.

```bash
grep -R "nc -l" /

```

Answer: **systemd service**

### Question 11: What is the absolute path of the malicious service?

Find the absolute path from the output of the `grep` command above.

Answer: **/etc/systemd/system/socket.service**

---

## Task 4: Even More Persistence

### Question 12: Which port is blocked on the victim’s firewall?

Use `iptables` to examine the firewall configuration.

```bash
iptables -L

```

Answer: **3578**

### Question 13: How do the firewall rules persist across reboots?

Check the root user's bash configuration files for persistence mechanisms.

```bash
cat /root/.bashrc

```

Answer: **/root/.bashrc**

### Question 14: How is the backdoored local Linux user named?

Inspect the `/etc/passwd` file for unusual user entries.

```bash
grep "/bin/bash" /etc/passwd

```

Answer: **support**

### Question 15: Which privileged group was assigned to the user?

Use the `groups` command to list group memberships for the user.

```bash
groups support

```

Answer: **sudo**

### Question 16: What is the strange word on one of the backdoored SSH keys?

View the `authorized_keys` file in the root user’s SSH directory.

```bash
cat /root/.ssh/authorized_keys

```

Answer: **ntsvc**

### Question 17: Can you spot and name one more popular persistence method? Not a MITRE technique name.

Check for files with the SUID bit set.

```bash
find / -perm -u=s -type f 2>/dev/null

```

Answer: **SUID binary**

### Question 18: What are the original and the backdoored binaries from question 6?

Verify the integrity of the suspected binary.

```bash
ls -l /usr/bin/clamav
dpkg --verify clamav

```

Answer: **/usr/bin/bash, /usr/bin/clamav**

### Question 19: What technique was used to hide the backdoor creation date?

Identify timestamp modification.

Answer: **Timestomping**

---

## Task 5: Final Target

### Question 20: What file was dropped which contained gathered victim information?

Check root’s `.bash_history` for evidence of dropped files.

```bash
cat /root/.bash_history

```

Answer: **/root/.dump.json**

### Question 21: According to the dropped dump, what is the server’s kernel version?

Decode and inspect `.dump.json` for details.

```bash
cat /root/.dump.json | base64 -d

```

Answer: **5.15.0–78-generic**

### Question 22: Which active internal IPs were found by the “rooter2” network scan?

Identify internal IPs from the dump.

Answer: **192.168.0.21,192.168.0.22**

### Question 23: How did the hacker find an exposed HTTP index on another internal IP?

Check the history for a network scan command.

```bash
grep -i "nc -zv" /root/.bash_history

```

Answer: **nc -zv 192.168.0.22 1024-10000 2>&1 | grep -v failed**

### Question 24: What command was used to exfiltrate the CDE database from the internal IP?

Locate the `wget` command in the history.

```bash
grep "wget" /root/.bash_history

```

Answer: **wget 192.168.0.22:8080/cde-backup.csv**

### Question 25: What is the most secret and precious string stored in the exfiltrated database?

Inspect the contents of the exfiltrated `.review.csv` file.

```bash
cat .review.csv | head -n 10

```

Answer: **pwned{v3ry-secur3-cardh0ld3r-data-environm3nt}**

---

Another hack from the Rubixverse…
