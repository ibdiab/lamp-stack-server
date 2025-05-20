# lamp-stack-server

This project demonstrates the setup and deployment of a LAMP stack (Linux, Apache, MariaDB, PHP) on Ubuntu, along with installing and configuring WordPress. This also includes server hardening and firewall configuration to protect your web server from hackers. **Note: This works with other Debian-based distributions and not just Ubuntu.**

---

# Step 1: Use a Static IP on Your Ubuntu System

Edit `/etc/netplan/99_config.yaml` using your preferred text editor. The reason behind a static IP is to ensure stability. When we have one consistent IP, the server knows to use that singular IP. However, if we use DHCP, that IP could change randomly. This results in several things breaking, like: firewall rules, hostnames, configs, etc. Your network interface might be different depending on your system. To identify your interface and default gateway:

```bash
ip a          # Find your interface name
ip route      # Find your default gateway
```

Replace `enp1s0` and `192.168.245.1` accordingly. Here’s a sample config:

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
sudo mysql_secure_installation
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

Use `nftables` to configure your firewall securely.

```bash
sudo systemctl stop ufw
sudo systemctl disable ufw
sudo systemctl start nftables
sudo systemctl enable nftables

sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p udp --dport 53 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

sudo iptables-save
```

✅ This allows HTTP, SSH, DNS, and apt functionality — while dropping everything else. This ensures the best functionality while retaining security. Confirm your rules with `sudo iptables -L -n -v`

---

# Step 9: Making Your First Blog Post

Now that WordPress is installed and secured, log in and create your first website, blog, or any web platform you like. You can use the built-in editor to:
- Share what you've learned
- Share tips or make tutorials
- Test some plugins and themes
Here's a sample entry I made explaining what a LAMP stack is along with some issues I encountered and how I fixed them: ![sample blog](https://github.com/user-attachments/assets/d4ff7477-157e-4f4f-b2ca-9e6969b99a40)
