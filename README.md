# lamp-stack-server

**For security purposes, please read this issue https://github.com/ibdiab/lamp-stack-server/issues/1 regarding 2 critical vulnerabilities for PHP, a key component of this LAMP stack setup.**

This project demonstrates the setup and deployment of a LAMP stack (Linux, Apache, MariaDB, PHP) on Ubuntu, along with installing and configuring WordPress. This also includes server hardening and firewall configuration to protect your web server from hackers. 

**Note: This works with every distribution based on Debian (ex. Ubuntu, Pop!_OS, Linux Mint, etc.)** I plan to add Fedora/Arch support but there are differences between them regarding package/service names, config paths, etc. that haven't been tested with this particular LAMP setup.
Feel free to test this and open a pull request with any results you find if you get there first.

For distributions outside of Fedora, Arch and Debian, please look into the packages your distro provides and take action accordingly. If your system doesn't use `systemctl` for service management, please look into the service management for your init system. I'm not familiar with distros outside of those 3, neither am I familiar with init systems outside of systemd. Feel free to open pull requests if you get this working on other distributions and init systems.

---

# Step 1: Use a Static IP on Your System
**Note: This step uses netplan, which is the default tool on Ubuntu for setting up networking. Other Debian/Ubuntu based distributions might not use this. In that case, use your distros own networking tool, then continue from Step 2.
**
Edit `/etc/netplan/99_config.yaml` using your preferred text editor. The reason behind a static IP is to ensure stability. When we have one consistent IP, the server knows to use that singular IP. However, if we use DHCP, that IP could change randomly. This results in several things breaking, like: firewall rules, hostnames, configs, etc. Your network interface might be different depending on your system. To identify your interface and default gateway:

```bash
ip a          # Find your interface name
ip route      # Find your default gateway
```

Replace `enp1s0` and `192.168.245.1` accordingly. `192.168.245.1` in this case is the first usable IP on your subnet, while `192.168.245` for example is your computers IP. Here’s a sample config:

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

Add the following line (adjust your username accordingly):

```
192.168.245.5 username
```

Test your internet and internal hostname resolution:

```bash
ping username
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
sudo apt install apache2 php php-mysql mariadb-server wordpress nftables
```

Descriptions:
- `apache2`: Web server
- `php`: Server software used with apache to render more complex websites
- `php-mysql`: PHP extension for MySQL/MariaDB
- `mariadb-server`: Database engine
- `wordpress`: Website/blog CMS
- `nftables`: Modern firewall utility

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

✏️ **Write your root password down somewhere safe.**

---

# Step 6: Setup MariaDB for WordPress

Connect to MySQL:

```bash
mysql -h localhost -u root -p  // Replace 'root' with your chosen admin username if different
```

SQL commands to configure your database and user:

```sql

CREATE DATABASE myblog;  // Replace 'myblog' with your blog database name

CREATE USER 'username'@'localhost' IDENTIFIED BY 'password';  // Replace 'username' and 'password' to match your setup

GRANT ALL PRIVILEGES ON myblog.* TO username@localhost IDENTIFIED BY 'password';  // Replace 'myblog', 'username', and 'password' to match your setup

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
sudo nano /etc/wordpress/config-username.php  // Replace 'username' with your actual computer's username
```

The PHP file allows WordPress to connect to the MariaDB database and find important files like plugins. Without this, we can't run the installer, our blog won't load, and WordPress won't be able to connect to the MariaDB database. Replace with your values:

```php
<?php
define('DB_NAME', 'myblog');  // Replace 'myblog' with your actual database name
define('DB_USER', 'username');  // Replace 'username' with your DB username
define('DB_PASSWORD', 'password');  // Replace 'password' with your DB password
define('DB_HOST', 'localhost');
define('WP_CONTENT_DIR', '/usr/share/wordpress/wp-content');
?>
```

Visit: `http://username/blog/wp-admin/install.php  // Replace 'username' with your system hostname` and follow the setup wizard.

---

# Step 8: System Hardening for Increased Protection

Use `UFW (Uncomplicated Firewall)` to configure your firewall securely.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw enable
```

✅ This allows HTTP, SSH, DNS, and apt functionality — while dropping everything else. This ensures the best functionality while retaining security. Confirm your rules with `sudo ufw status verbose`

---

# Step 9: Making Your First Blog Post

Now that WordPress is installed and secured, log in and create your first website, blog, or any web platform you like. You can use the built-in editor to:
- Share what you've learned
- Share tips or make tutorials
- Test some plugins and themes

Here's a sample blog post I made explaining what a LAMP stack is along with some issues I encountered and how I fixed them: ![sample blog](https://github.com/user-attachments/assets/d4ff7477-157e-4f4f-b2ca-9e6969b99a40)
