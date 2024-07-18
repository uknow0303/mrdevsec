### Initial Setup and Planning

**1. Inventory and Purpose of Each Machine:**

- **RHEL Laptop:** Will act as the DNS server and domain controller.
- **Kali Linux Laptop:** Used for security testing and bug bounty tasks.
- **Proxmox VE Server:** For creating and managing virtual machines.
- **Windows Laptop:** Everyday use and possibly domain-joined for testing.
- **Desktop PC:** To be used for software installation, potentially a web server.

**2. Network Topology and IP Planning:**

- Plan out your IP address ranges (e.g., 192.168.1.0/24 for the main network, 192.168.2.0/24 for the guest network, etc.).
- Decide on a subnetting scheme if needed to separate different parts of your network (e.g., management, servers, clients).

**3. Security Considerations and Policies:**

- Define security policies such as password complexity, user roles, and permissions.
- Plan for physical security measures (e.g., secure server room, locks).
- Implement a regular update and patch management schedule.

### Next Steps

1. **Setting up the Router and Firewall:**
    - Configure your router to manage network traffic and secure it with a strong password.
    - Enable the firewall and set up basic rules to block unwanted traffic.
    - Consider setting up VLANs to segment different parts of your network (e.g., separate network for guests, servers, and workstations).
2. **Configuring DNS and DHCP:**
    - On your RHEL laptop, install and configure DNS services using `BIND`.
    - Set up DHCP on your router or another dedicated machine to assign IP addresses dynamically.
    - Ensure that DNS is properly resolving and that DHCP is handing out IP addresses correctly.

### Setting Up the Router and Firewall

**1. Configuring Your Router:**

- **Access Router Settings:**
    - Connect to your router's admin interface (usually through a web browser at an IP like 192.168.1.1).
    - Log in using the admin credentials.
- **Basic Configuration:**
    - Change the default admin password to something strong and unique.
    - Set up the basic network settings, such as the LAN IP address (e.g., 192.168.1.1).
- **Enable DHCP:**
    - If your router will handle DHCP, enable the DHCP server and configure the IP address range (e.g., 192.168.1.100 to 192.168.1.200).
- **Wi-Fi Settings:**
    - Configure SSIDs for your Wi-Fi networks.
    - Set strong WPA3 or WPA2 encryption with a secure passphrase.
    - Optionally, set up a separate guest network with its own SSID.
- **Firewall and Security:**
    - Enable the firewall feature on your router.
    - Disable remote management unless absolutely necessary.
    - Configure port forwarding rules only if needed for specific applications.

**2. Setting Up VLANs for Network Segmentation:**

- **Create VLANs:**
    - Access the VLAN settings on your router.
    - Create VLANs for different parts of your network (e.g., VLAN 10 for management, VLAN 20 for servers, VLAN 30 for workstations, VLAN 40 for guests).
- **Assign Ports to VLANs:**
    - Assign the router's ports to the appropriate VLANs.
    - Configure VLAN tagging if your router supports it, especially if using managed switches downstream.
- **Configure Inter-VLAN Routing:**
    - Set up routing rules to allow necessary communication between VLANs while restricting unnecessary access.

**3. Implementing Firewall Rules:**

- **Basic Rules:**
    - Allow traffic within your internal network.
    - Allow outbound traffic to the internet.
    - Block all inbound traffic from the internet by default.
- **Specific Rules:**
    - Allow necessary inbound traffic for services you run (e.g., a web server on port 80 or 443).
    - Set up rules to block traffic between certain VLANs if needed (e.g., block guest VLAN from accessing the server VLAN).

**Example Basic Firewall Rules:**

```
# Allow all traffic within the internal network
allow from 192.168.1.0/24 to 192.168.1.0/24

# Allow outbound traffic to the internet
allow from 192.168.1.0/24 to any

# Block all inbound traffic from the internet
block from any to 192.168.1.0/24

# Allow inbound HTTP/HTTPS traffic to web server
allow from any to 192.168.1.100 port 80,443

```

**4. Securing Your Router:**

- **Firmware Updates:**
    - Regularly check for and install firmware updates for your router to ensure you have the latest security patches.
- **Monitoring and Logs:**
    - Enable logging features on your router to monitor traffic and detect any suspicious activity.
    - Review logs regularly.

With the router and basic network setup completed, we can now move on to configuring the DNS and DHCP services. (RHEL)

### Setting Up DNS and DHCP on RHEL

**1. Installing and Configuring DNS (BIND):**

- **Install BIND:**
    
    ```bash
    sudo yum install bind bind-utils
    
    ```
    
- **Configure BIND:**
    - Edit the BIND configuration file `/etc/named.conf`:
        
        ```
        options {
            listen-on port 53 { 127.0.0.1; any; };
            allow-query     { any; };
            recursion yes;
            ...
        };
        
        zone "example.com" IN {
            type master;
            file "example.com.db";
            allow-update { none; };
        };
        
        ```
        
    - Create a zone file for your domain (e.g., `/var/named/example.com.db`):
        
        ```
        $TTL 86400
        @   IN  SOA ns1.example.com. admin.example.com. (
                    2024052901 ; Serial
                    3600       ; Refresh
                    1800       ; Retry
                    604800     ; Expire
                    86400 )    ; Minimum TTL
        
            IN  NS  ns1.example.com.
        ns1 IN  A   192.168.1.100
        
        ```
        
- **Start and Enable BIND:**
    
    ```bash
    sudo systemctl start named
    sudo systemctl enable named
    
    ```
    
- **Open Firewall Ports for DNS:**
    
    ```bash
    sudo firewall-cmd --permanent --add-port=53/tcp
    sudo firewall-cmd --permanent --add-port=53/udp
    sudo firewall-cmd --reload
    
    ```
    

**2. Installing and Configuring DHCP:**

- **Install DHCP Server:**
    
    ```bash
    sudo yum install dhcp
    
    ```
    
- **Configure DHCP:**
    - Edit the DHCP configuration file `/etc/dhcp/dhcpd.conf`:
        
        ```
        subnet 192.168.1.0 netmask 255.255.255.0 {
            range 192.168.1.100 192.168.1.200;
            option routers 192.168.1.1;
            option domain-name-servers 192.168.1.100;
            option domain-name "example.com";
        }
        
        ```
        
- **Start and Enable DHCP:**
    
    ```bash
    sudo systemctl start dhcpd
    sudo systemctl enable dhcpd
    
    ```
    
- **Open Firewall Port for DHCP:**
    
    ```bash
    sudo firewall-cmd --permanent --add-service=dhcp
    sudo firewall-cmd --reload
    
    ```
    

### Setting Up Red Hat Enterprise Linux (RHEL) as a Domain Controller with Windows Active Directory Integration

To set up RHEL as a domain controller and integrate it with Windows Active Directory, we will use Samba, which provides file and print services and can act as an Active Directory Domain Controller.

**1. Install Samba and Required Packages:**

- **Install Samba:**
    
    ```bash
    sudo yum install samba samba-client samba-common samba-dc samba-dc-libs
    
    ```
    
- **Install Kerberos and other necessary packages:**
    
    ```bash
    sudo yum install krb5-workstation krb5-libs krb5-auth-dialog
    sudo yum install ntp
    
    ```
    

**2. Configure Samba:**

- **Set up Samba AD Directory:**
    
    ```bash
    sudo samba-tool domain provision --use-rfc2307 --interactive
    
    ```
    
    During the interactive setup, you will need to provide:
    
    - Domain (e.g., `EXAMPLE`)
    - Realm (e.g., `EXAMPLE.COM`)
    - Server role (should be set to `dc`)
    - DNS backend (typically `SAMBA_INTERNAL`)
    - Administrator password
- **Edit the `/etc/krb5.conf` file:**
    
    ```
    [logging]
      default = FILE:/var/log/krb5libs.log
      kdc = FILE:/var/log/krb5kdc.log
      admin_server = FILE:/var/log/kadmind.log
    
    [libdefaults]
      default_realm = EXAMPLE.COM
      dns_lookup_realm = false
      dns_lookup_kdc = true
    
    [realms]
      EXAMPLE.COM = {
        kdc = your_RHEL_server_ip
        admin_server = your_RHEL_server_ip
      }
    
    [domain_realm]
      .example.com = EXAMPLE.COM
      example.com = EXAMPLE.COM
    
    ```
    
- **Edit the `/etc/samba/smb.conf` file:**
This should already be configured by the Samba provision script, but ensure it contains:
    
    ```
    [global]
      workgroup = EXAMPLE
      realm = EXAMPLE.COM
      netbios name = your_RHEL_hostname
      server role = active directory domain controller
      dns forwarder = your_dns_forwarder_ip
    
    ```
    

**3. Start Samba Services:**

- **Enable and Start Samba:**
    
    ```bash
    sudo systemctl enable samba
    sudo systemctl start samba
    
    ```
    
- **Open Firewall Ports:**
Samba requires several ports to be open for proper AD operation:
    
    ```bash
    sudo firewall-cmd --permanent --add-service=samba
    sudo firewall-cmd --permanent --add-port=53/tcp
    sudo firewall-cmd --permanent --add-port=53/udp
    sudo firewall-cmd --permanent --add-port=88/tcp
    sudo firewall-cmd --permanent --add-port=88/udp
    sudo firewall-cmd --permanent --add-port=135/tcp
    sudo firewall-cmd --permanent --add-port=137/udp
    sudo firewall-cmd --permanent --add-port=138/udp
    sudo firewall-cmd --permanent --add-port=139/tcp
    sudo firewall-cmd --permanent --add-port=389/tcp
    sudo firewall-cmd --permanent --add-port=389/udp
    sudo firewall-cmd --permanent --add-port=445/tcp
    sudo firewall-cmd --permanent --add-port=464/tcp
    sudo firewall-cmd --permanent --add-port=464/udp
    sudo firewall-cmd --permanent --add-port=636/tcp
    sudo firewall-cmd --permanent --add-port=3268/tcp
    sudo firewall-cmd --permanent --add-port=3269/tcp
    sudo firewall-cmd --reload
    
    ```
    

**4. Configure NTP (Network Time Protocol):**

- **Edit `/etc/ntp.conf`:**
Ensure you have the correct NTP servers configured:
    
    ```
    server 0.centos.pool.ntp.org iburst
    server 1.centos.pool.ntp.org iburst
    server 2.centos.pool.ntp.org iburst
    server 3.centos.pool.ntp.org iburst
    
    ```
    
- **Enable and Start NTP:**
    
    ```bash
    sudo systemctl enable ntpd
    sudo systemctl start ntpd
    
    ```
    

**5. Join Windows Clients to the Domain:**

- **Windows Settings:**
    - Open `Control Panel` > `System and Security` > `System`.
    - Click on `Change settings` next to the computer name.
    - Click on `Change` and select `Domain`, then enter your domain name (e.g., `example.com`).
    - Enter the credentials of the Samba AD administrator.
    - Restart the Windows machine.

**6. Manage Users and Groups:**

- **Add Users and Groups using Samba Tool:**
    
    ```bash
    sudo samba-tool user add username
    sudo samba-tool group add groupname
    sudo samba-tool group addmembers groupname username
    
    ```
    

### Setting Up Network Storage

To set up network storage, we'll configure a Network Attached Storage (NAS) solution. For this setup, we can use the Proxmox VE server to create a virtual machine (VM) dedicated to serving as the NAS. We'll use OpenMediaVault (OMV), a popular and user-friendly NAS operating system.

**1. Preparing Proxmox VE:**

- **Create a New VM for OMV:**
    1. Log in to the Proxmox web interface.
    2. Click on `Create VM`.
    3. Fill in the `General` settings:
        - VM ID: Choose an ID or leave it as default.
        - Name: Name your VM (e.g., `OMV-NAS`).
    4. Configure the `OS` settings:
        - Use an appropriate ISO image for OMV (you can download it from the OMV website and upload it to Proxmox).
    5. Configure `System` settings:
        - BIOS: Default (SeaBIOS).
        - Machine: Default (i440fx).
    6. Configure `Hard Disk` settings:
        - Bus/Device: VirtIO Block.
        - Storage: Select the storage where the VM disk will reside.
        - Disk size: Allocate sufficient disk space for the OMV system.
    7. Configure `CPU` settings:
        - Cores: Allocate the number of cores based on your server’s capacity.
    8. Configure `Memory` settings:
        - Allocate the necessary RAM for OMV.
    9. Configure `Network` settings:
        - Bridge: Select the appropriate bridge (e.g., `vmbr0`).
    10. Review and confirm the settings, then click `Finish`.
- **Install OpenMediaVault on the VM:**
    1. Start the VM and open the console.
    2. Follow the OMV installation instructions, setting up the root password and configuring basic network settings.
    3. Complete the installation and reboot the VM.

**2. Configuring OpenMediaVault:**

- **Access the OMV Web Interface:**
    - Open a web browser and navigate to `http://<OMV_VM_IP_Address>`.
    - Log in using the default credentials (username: `admin`, password: `openmediavault`).
- **Initial Configuration:**
    - Change the admin password to something secure.
    - Set the time zone and other general settings.

**3. Setting Up Storage:**

- **Add Physical Disks:**
    - Go to `Storage` > `Disks`.
    - Ensure your physical disks are recognized by OMV. If not, make sure they are correctly attached to the VM.
- **Create a Filesystem:**
    - Go to `Storage` > `File Systems`.
    - Click on `Create`, select the disk, and choose the filesystem type (ext4 is recommended).
    - Mount the new filesystem.

**4. Configuring Shared Folders:**

- **Create Shared Folders:**
    - Go to `Access Rights Management` > `Shared Folders`.
    - Click `Add` to create a new shared folder.
    - Select the device, path, and name for the shared folder.
- **Set Permissions:**
    - Assign appropriate permissions for the shared folder based on your user needs.

**5. Setting Up Network Shares:**

- **Enable SMB/CIFS (for Windows):**
    - Go to `Services` > `SMB/CIFS`.
    - Enable the SMB/CIFS service.
    - Configure the workgroup and other settings as needed.
    - Click `Save` and `Apply`.
- **Add SMB/CIFS Share:**
    - Go to `Services` > `SMB/CIFS` > `Shares`.
    - Click `Add` to create a new share.
    - Select the shared folder you created earlier.
    - Configure the share settings (e.g., public/private, guest access).
    - Click `Save` and `Apply`.

**6. Configuring User Accounts:**

- **Create Users:**
    - Go to `Access Rights Management` > `User`.
    - Click `Add` to create a new user.
    - Assign the user to groups and configure their permissions.

**7. Accessing the Network Share:**

- **From Windows:**
    - Open `File Explorer`.
    - Enter `\\\\<OMV_VM_IP_Address>\\<Share_Name>` in the address bar.
    - Enter the credentials if required and access the shared folder.
- **From Linux:**
    - Use the following command to mount the SMB share:
        
        ```bash
        sudo mount -t cifs //<OMV_VM_IP_Address>/<Share_Name> /mnt/your_mount_point -o username=your_username,password=your_password
        
        ```
        

### Setting Up Nextcloud for Personal Cloud Storage

Nextcloud is an excellent choice for personal cloud storage. We will set it up on a virtual machine (VM) on your Proxmox VE server and use Cloudflare for DNS and SSL/TLS management.

**1. Prepare the Proxmox VE Environment:**

- **Create a New VM for Nextcloud:**
    1. Log in to the Proxmox web interface.
    2. Click on `Create VM`.
    3. Fill in the `General` settings:
        - VM ID: Choose an ID or leave it as default.
        - Name: Name your VM (e.g., `Nextcloud`).
    4. Configure the `OS` settings:
        - Use an appropriate ISO image for Ubuntu Server (you can download it from the Ubuntu website and upload it to Proxmox).
    5. Configure `System` settings:
        - BIOS: Default (SeaBIOS).
        - Machine: Default (i440fx).
    6. Configure `Hard Disk` settings:
        - Bus/Device: VirtIO Block.
        - Storage: Select the storage where the VM disk will reside.
        - Disk size: Allocate sufficient disk space for Nextcloud.
    7. Configure `CPU` settings:
        - Cores: Allocate the number of cores based on your server’s capacity.
    8. Configure `Memory` settings:
        - Allocate the necessary RAM for Nextcloud.
    9. Configure `Network` settings:
        - Bridge: Select the appropriate bridge (e.g., `vmbr0`).
    10. Review and confirm the settings, then click `Finish`.
- **Install Ubuntu Server on the VM:**
    1. Start the VM and open the console.
    2. Follow the Ubuntu Server installation instructions, setting up the root password and configuring basic network settings.
    3. Complete the installation and reboot the VM.

**2. Installing Nextcloud:**

- **Update and Upgrade the System:**
    
    ```bash
    sudo apt update
    sudo apt upgrade -y
    
    ```
    
- **Install Necessary Packages:**
    
    ```bash
    sudo apt install apache2 mariadb-server libapache2-mod-php7.4
    sudo apt install php7.4-gd php7.4-json php7.4-mysql php7.4-curl php7.4-mbstring
    sudo apt install php7.4-intl php7.4-xml php7.4-zip php7.4-ldap php7.4-imap
    sudo apt install php7.4-apcu php7.4-redis php7.4-igbinary php7.4-bz2
    sudo apt install php7.4-gmp php-apcu redis-server
    
    ```
    
- **Download and Extract Nextcloud:**
    
    ```bash
    wget <https://download.nextcloud.com/server/releases/nextcloud-21.0.1.zip>
    unzip nextcloud-21.0.1.zip
    sudo mv nextcloud /var/www/
    
    ```
    
- **Set Directory Permissions:**
    
    ```bash
    sudo chown -R www-data:www-data /var/www/nextcloud/
    sudo chmod -R 755 /var/www/nextcloud/
    
    ```
    
- **Configure Apache:**
    - Create a configuration file for Nextcloud:
    Add the following content:
        
        ```bash
        sudo nano /etc/apache2/sites-available/nextcloud.conf
        
        ```
        
        ```
        <VirtualHost *:80>
            DocumentRoot /var/www/nextcloud/
            ServerName your-domain.com
        
            <Directory /var/www/nextcloud/>
                Options +FollowSymlinks
                AllowOverride All
        
                <IfModule mod_dav.c>
                    Dav off
                </IfModule>
        
                SetEnv HOME /var/www/nextcloud
                SetEnv HTTP_HOME /var/www/nextcloud
            </Directory>
        
            ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
            CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
        </VirtualHost>
        
        ```
        
    - Enable the configuration and necessary Apache modules:
        
        ```bash
        sudo a2ensite nextcloud.conf
        sudo a2enmod rewrite headers env dir mime setenvif
        sudo systemctl restart apache2
        
        ```
        

**3. Configuring the Database:**

- **Secure MariaDB Installation:**
    
    ```bash
    sudo mysql_secure_installation
    
    ```
    
- **Create Database and User for Nextcloud:**
    
    ```bash
    sudo mysql -u root -p
    
    ```
    
    In the MariaDB prompt:
    
    ```sql
    CREATE DATABASE nextcloud;
    CREATE USER 'nextclouduser'@'localhost' IDENTIFIED BY 'yourpassword';
    GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextclouduser'@'localhost';
    FLUSH PRIVILEGES;
    EXIT;
    
    ```
    

**4. Completing Nextcloud Setup:**

- **Finish Installation via Web Interface:**
    - Open a web browser and navigate to `http://your-domain.com`.
    - Follow the on-screen instructions to complete the setup, entering the database details created earlier.

**5. Setting Up SSL with Cloudflare:**

- **Update DNS Settings in Cloudflare:**
    - Log in to your Cloudflare account.
    - Add an A record for your domain pointing to the public IP of your Nextcloud server.
- **Configure SSL/TLS:**
    - In Cloudflare, go to the SSL/TLS section.
    - Set the SSL mode to `Full` (strict).
    - Obtain and configure the Cloudflare Origin Certificate:
    Add the following content:
        
        ```bash
        sudo nano /etc/apache2/sites-available/nextcloud-ssl.conf
        
        ```
        
        ```
        <VirtualHost *:443>
            DocumentRoot /var/www/nextcloud/
            ServerName your-domain.com
        
            <Directory /var/www/nextcloud/>
                Options +FollowSymlinks
                AllowOverride All
        
                <IfModule mod_dav.c>
                    Dav off
                </IfModule>
        
                SetEnv HOME /var/www/nextcloud
                SetEnv HTTP_HOME /var/www/nextcloud
            </Directory>
        
            SSLEngine on
            SSLCertificateFile /etc/ssl/certs/cloudflare-origin-pem.crt
            SSLCertificateKeyFile /etc/ssl/private/cloudflare-origin-pem.key
        
            ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
            CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
        </VirtualHost>
        
        ```
        
    - Enable the SSL site and reload Apache:
        
        ```bash
        sudo a2ensite nextcloud-ssl.conf
        sudo systemctl reload apache2
        
        ```
        

### More Stuff I need to do

### 1. **Backup and Redundancy Planning:**

Ensuring your data is backed up and recoverable is crucial, especially given the sensitivity of the information you handle.

- **Set Up Automated Backups:**
    - Configure automated backups for your Nextcloud data and database. You can use tools like `rsync`, `borgbackup`, or Nextcloud's own backup solutions.
    - Set up database backups using `mysqldump` and store these backups securely.
- **Offsite Backups:**
    - Consider implementing offsite backups using cloud storage solutions or a secondary physical location.

### 2. **Security Enhancements:**

- **Implement Two-Factor Authentication (2FA):**
    - Set up 2FA for Nextcloud and any other critical services to add an extra layer of security.
- **Regular Security Audits:**
    - Conduct regular security audits and vulnerability assessments on your network and services. Tools like OpenVAS, Nessus, or the built-in tools in Kali Linux can be helpful.

### 3. **Monitoring and Logging:**

- **Centralized Logging:**
    - Set up a centralized logging system using tools like the ELK stack (Elasticsearch, Logstash, Kibana) or Graylog. This will help you monitor and analyze logs from different services and machines.
- **Network Monitoring:**
    - Implement network monitoring to keep an eye on the traffic and detect any suspicious activity. Tools like Nagios, Zabbix, or Prometheus can be useful for this.

### 4. **Virtual Environment Expansion:**

- **Create Additional VMs:**
    - Use Proxmox VE to create additional VMs for various purposes, such as testing new applications, simulating different network configurations, or hosting other services.

### 5. **Improve Network Performance and Reliability:**

- **Upgrade Network Hardware:**
    - Consider upgrading your network hardware (routers, switches) to support higher speeds and better reliability.
    - Implement redundancy for critical network components to avoid single points of failure.

### 6. **User Training and Documentation:**

- **User Training:**
    - Provide training for users on best practices for security, data handling, and using the new systems.
- **Documentation:**
    - Maintain detailed documentation of your setup, configurations, and procedures. This will help with troubleshooting and future expansions.

### 7. **Additional Services:**

- **VPN Setup:**
    - Set up a VPN to allow secure remote access to your internal network.
- **Web Application Firewall (WAF):**
    - Implement a WAF to protect your web services from common attacks.
