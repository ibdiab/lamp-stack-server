# lamp-stack-server

**For security purposes, please read this issue https://github.com/ibdiab/lamp-stack-server/issues/1 regarding 2 critical vulnerabilities for PHP, a key component of this LAMP stack setup.**

This project demonstrates the setup and deployment of a LAMP stack (Linux, Apache, MariaDB, PHP) on Ubuntu, along with installing and configuring WordPress. This also includes server hardening and firewall configuration to protect your web server from hackers.

**Note: This should work with most distributions based on Debian (ex. Ubuntu, Pop!_OS, Linux Mint, etc.)**
There are a few steps that are Ubuntu specific (ex. netplan). I ran this LAMP stack on Ubuntu 26.04 LTS. Other Debian/Ubuntu based Linux flavors might not use this. In that case, follow your distributions guide to set a static IP on your server and continue from step 2.

Feel free to test this and open a pull request with any results you find if you get there first.

---

# **Architecture Diagram**

This diagram provides a visual look into how network traffic reaches the server. Any client on the internet and your personal computer (used to connect to the server) go through the firewall. Only HTTP (80) is allowed in, and all other ports are blocked. Optionally, HTTPS (443) and SSH (22) can be allowed if needed: 
<img width="1920" height="1200" alt="Architecture Diagram" src="https://github.com/user-attachments/assets/af263af4-f32b-444d-bf46-b49bf4190dae" />



---

# Step 1: Use a Static IP on Your Server
**Note:** This step uses netplan, which is the default tool on Ubuntu for setting up networking. Other Debian/Ubuntu based distributions might not use this. In that case, use your distros own networking tool, then continue from Step 2.


Edit `/etc/netplan/99_config.yaml` using your preferred text editor. The reason behind a static IP is to ensure stability. When we have one consistent IP, the server knows to use that singular IP. However, if we use DHCP, that IP could change randomly. This results in several things breaking, like: firewall rules, hostnames, configs, etc. Your network interface might be different depending on your system. To identify your interface and default gateway:

```bash
ip a          # Find your interface name
ip route      # Find your default gateway
```

Replace `enp1s0` and `192.168.245.1` accordingly. `192.168.245.1` in this case is the first usable IP on your subnet, while `192.168.245.5` for example is your server's IP. Here’s a sample config:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0:
      addresses:
        - 192.168.245.5/24
      routes:
        - to: default
          via: 192.168.245.1
      nameservers:
        addresses: [192.168.245.1]
```

Apply permissions to the file such that only the owner can read and write it, no one else can modify it:
```bash
sudo chmod 600 /etc/netplan/99_config.yaml
```

Apply the configuration:

```bash
sudo netplan apply
```

---

# Step 2: (Optional but highly recommended) Setup Local Hostname Resolution

This step makes WordPress configuration easier, since you won't need to type the full IP address in order to access WordPress once configuration is done. Edit `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

Add the following line (adjust your IP and hostname accordingly):

```
192.168.245.5 wpserver
```

Test your internet and internal hostname resolution:

```bash
ping wpserver
ping www.debian.org
```

---

# Step 3: Installing Packages

Update and upgrade your system:

```bash
sudo apt update && sudo apt upgrade
```

Install the required packages:

```bash
sudo apt install apache2 php php-mysql mariadb-server wordpress
```

Descriptions:
- `apache2`: Web server
- `php`: Server software used with apache to render more complex websites
- `php-mysql`: PHP extension for MySQL/MariaDB
- `mariadb-server`: Database engine
- `wordpress`: Website/blog CMS

---

# Step 4: Configuring Apache

Check Apache's status:

```bash
sudo systemctl status apache2
```

Enable it on boot if necessary:

```bash
sudo systemctl enable apache2
```

Restart and test:

```bash
sudo systemctl restart apache2
```
Visit `http://<your-ip>` in a browser — you should see the Apache test page.

---

# Step 5: Installing MariaDB

Check MariaDB's status:

```bash
sudo systemctl status mariadb
```

Enable it on boot if needed:

```bash
sudo systemctl enable mariadb
```

Run the secure installation script:

```bash
sudo mariadb-secure-installation
```

Follow the prompts to:
- Set the root password
- Remove anonymous users
- Disable root login remotely
- Remove the test database
- Reload the privilege tables

✏️ **Write your root password down somewhere safe, and ensure it is a secure password.**

---

# Step 6: Setup MariaDB for WordPress

Connect to MariaDB:

```bash
mysql -h localhost -u root -p  # Replace 'root' with your chosen admin username if different
```

SQL commands to configure your database and user:

```sql

CREATE DATABASE wordpressDB;  # Replace 'wordpressDB' with your database's name

CREATE USER 'username'@'localhost' IDENTIFIED BY 'password';  # Replace 'username' and 'password' to match your setup

GRANT ALL PRIVILEGES ON wordpressDB.* TO username@localhost IDENTIFIED BY 'password';  # Replace 'wordpressDB', 'username', and 'password' to match your setup

FLUSH PRIVILEGES;
```

Then exit:

```sql
EXIT;
```

---

# Step 7: WordPress Configuration

Create the Apache virtual host file:

```bash
sudo nano /etc/apache2/sites-available/wordpress.conf
```

The following code will relate our blog to the directory `/usr/share/wordpress` and grants access to WordPress content. Paste the following:

```apache
Alias /blog /usr/share/wordpress
<Directory /usr/share/wordpress>
    Options FollowSymLinks
    AllowOverride Limit Options FileInfo
    DirectoryIndex index.php
    Require all granted
</Directory>
<Directory /usr/share/wordpress/wp-content>
    Options FollowSymLinks
    Require all granted
</Directory>
```

Enable the site and restart Apache:

```bash
sudo a2ensite wordpress
sudo systemctl restart apache2
```

Create the WordPress config file:

```bash
sudo nano /etc/wordpress/config-wpserver.php  # Replace 'wpserver' with the hostname from your /etc/hosts file
```

The PHP file allows WordPress to connect to the MariaDB database and find important files like plugins. Without this, we can't run the installer, our blog won't load, and WordPress won't be able to connect to the MariaDB database. Replace with your values:

```php
<?php
define('DB_NAME', 'wordpressDB');  # Replace 'wordpressDB' with your database's name
define('DB_USER', 'username');  # Replace 'username' with your DB username
define('DB_PASSWORD', 'password');  # Replace 'password' with the DB password you set in Step 6
define('DB_HOST', 'localhost');
define('WP_CONTENT_DIR', '/usr/share/wordpress/wp-content');
?>
```

Visit `http://wpserver/blog/wp-admin/install.php` (replace `wpserver` with the hostname from your `/etc/hosts` file) and follow the setup wizard.

---

# Step 8: System Hardening for Increased Protection

Use `UFW (Uncomplicated Firewall)` to configure your firewall securely.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 80/tcp
sudo ufw enable
```

✅ This allows HTTP - while dropping all other incoming traffic. This ensures the best functionality while retaining security. Outgoing traffic is allowed, so the server can still use apt and DNS. Confirm your rules with `sudo ufw status verbose`

---

# Step 9: Making Your First Blog Post

Now that WordPress is installed and secured, log in and create your first website, blog, or any web platform you like. You can use the built-in editor to:
- Share what you've learned
- Share tips or make tutorials
- Test some plugins and themes

Here's a sample blog post I made explaining what a LAMP stack is along with some issues I encountered and how I fixed them: ![sample blog](https://github.com/user-attachments/assets/d4ff7477-157e-4f4f-b2ca-9e6969b99a40)

# *Optional* Step 10: Enabling HTTPS for better security and SSH for remote connections to the server

## HTTPS 
This uses Apache's built-in self-signed certificate, which I tested on Ubuntu 26.04 LTS. It encrypts traffic, but browsers can't verify who issued the certificate. A browser-trusted certificate (Let's Encrypt) requires a domain you own, which I haven't tested.

Run the below commands to set up HTTPS for your server:

```bash
sudo a2enmod ssl
sudo a2ensite default-ssl
sudo ufw allow 443/tcp
sudo systemctl restart apache2
```

Then go to `https://wpserver/blog/wp-login.php` (use the name from your `/etc/hosts` file). Your browser will warn that the certificate is self-signed. That's expected, since no trusted authority issued it. Only continue on your own network, and not if you see this warning on a server you didn't set up yourself.

## SSH
This is best used when you need to remotely connect to your server to make configuration changes. The commands below allow SSH from your LAN only, so it isn't exposed to the internet.

Install the SSH server if not already installed:

```bash
sudo apt install openssh-server
```

Find your LAN's subnet with `ip route` (the line like `192.168.245.0/24`), then allow SSH from it only. Replace the subnet with yours:

```bash
sudo ufw allow from 192.168.245.0/24 to any port 22 proto tcp
```

Ensure your client computer (the system you will use to remotely connect to the server), has the same `/etc/hosts` entry.

Confirm the rule with `sudo ufw status verbose`. Then, from another computer on your network, connect with:

```bash
ssh your-linux-username@wpserver
```

Replace `your-linux-username` with the account you log into the server with, and `wpserver` with the hostname from your `/etc/hosts` file (or use the server's IP address). This is where your Linux username matters, unlike the hostname and database names.
