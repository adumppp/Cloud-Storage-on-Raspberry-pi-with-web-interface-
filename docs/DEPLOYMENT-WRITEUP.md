# Deploying a Personal Cloud Server on Raspberry Pi

## Project overview

This project turns a Raspberry Pi 3 into a personal cloud-storage server using Nextcloud. The server provides a web interface for storing, downloading, organizing, and synchronizing files. Apache serves the website, PHP runs Nextcloud, and MariaDB stores application data.

The local-network deployment was completed and tested successfully. Public access through Cloudflare Tunnel is the planned second phase and is documented separately.

## Objectives

The project was designed to:

1. Install a Linux operating system on a Raspberry Pi.
2. Configure a web and database server.
3. Deploy Nextcloud as a private alternative to hosted storage services.
4. Access the server from computers and mobile devices on the same network.
5. Prepare the server for secure public access without opening router ports.

## System architecture

```text
Client browser
     │
     │ HTTP on the local network
     ▼
Raspberry Pi 3
     ├── Apache web server
     ├── PHP runtime
     ├── Nextcloud application
     ├── MariaDB database
     └── Local or USB file storage
```

For the planned public deployment:

```text
Internet client
     │
     │ HTTPS
     ▼
Cloudflare edge
     │
     │ Outbound encrypted tunnel
     ▼
cloudflared on Raspberry Pi ──> Apache ──> Nextcloud
```

## Requirements

### Hardware

- Raspberry Pi 3
- MicroSD card with at least 32 GB capacity
- Reliable Raspberry Pi power supply
- Monitor, keyboard, and mouse for initial setup, or SSH access
- Network connection
- Optional USB storage formatted with a Linux filesystem such as ext4

### Software

- Raspberry Pi OS
- Apache
- PHP and Nextcloud-required PHP modules
- MariaDB
- Nextcloud Server
- A modern web browser
- Cloudflare account and domain for the planned public phase

## Phase 1: Prepare Raspberry Pi OS

### Step 1 — Flash the operating system

1. Install Raspberry Pi Imager on another computer.
2. Insert the MicroSD card.
3. Select Raspberry Pi OS and the target card.
4. Configure the username, password, Wi-Fi, locale, and SSH settings if required.
5. Write the image and insert the card into the Raspberry Pi.
6. Boot the Pi and complete the first-start setup.

The original project used Raspberry Pi OS 32-bit. A new deployment should confirm that the selected Raspberry Pi OS, PHP version, and Nextcloud release are mutually supported before installation.

### Step 2 — Update the operating system

Open a terminal and run:

```bash
sudo apt update
sudo apt upgrade -y
sudo reboot
```

After rebooting, confirm the Pi is connected to the network:

```bash
hostname -I
```

Record the local address or configure a DHCP reservation in the router. The address can change if no reservation is used.

## Phase 2: Install the web stack

### Step 3 — Install Apache

```bash
sudo apt install apache2 -y
sudo systemctl enable --now apache2
sudo systemctl status apache2
```

Open the following address from the Pi or another computer on the same network:

```text
http://<PI-LAN-IP>/
```

The Apache default page should appear.

### Step 4 — Install PHP and required modules

```bash
sudo apt install php php-gd php-mbstring php-xml php-zip php-curl php-intl php-bz2 php-fpm php-mysql php-imagick -y
sudo systemctl restart apache2
php --version
```

Before installing a current Nextcloud version, compare the displayed PHP version with Nextcloud's current system requirements.

### Step 5 — Install MariaDB

```bash
sudo apt install mariadb-server mariadb-client -y
sudo systemctl enable --now mariadb
sudo systemctl status mariadb
```

If available on the installed MariaDB version, run its security wizard:

```bash
sudo mariadb-secure-installation
```

On Raspberry Pi OS, the MariaDB root account may authenticate through the local Unix socket. A separate database account should be used by Nextcloud.

## Phase 3: Configure the database

### Step 6 — Create the Nextcloud database and user

Open the MariaDB console:

```bash
sudo mariadb
```

Run the following SQL, replacing the password placeholder with a long, unique, randomly generated password:

```sql
CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'nextuser'@'localhost' IDENTIFIED BY 'REPLACE_WITH_A_RANDOM_PASSWORD';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Test the dedicated account:

```bash
mariadb -u nextuser -p nextcloud
```

Enter the new password, confirm that the connection works, and type `EXIT;`.

Never use the example password from an online tutorial or publish the real password in screenshots, reports, or Git repositories.

## Phase 4: Install Nextcloud

### Step 7 — Download and extract Nextcloud

The original deployment used the official latest-release archive:

```bash
cd /var/www
sudo wget https://download.nextcloud.com/server/releases/latest.zip
sudo apt install unzip -y
sudo unzip latest.zip
sudo chown -R www-data:www-data /var/www/nextcloud
```

Always download Nextcloud from the official Nextcloud domain. For a controlled production deployment, select a supported release deliberately and verify its published checksum instead of depending permanently on an unversioned download.

### Step 8 — Choose the data directory

The completed local project used:

```text
/var/www/nextcloud/data
```

For a new public-facing installation, it is safer to place user data outside the web root:

```bash
sudo mkdir -p /srv/nextcloud-data
sudo chown -R www-data:www-data /srv/nextcloud-data
sudo chmod 750 /srv/nextcloud-data
```

The chosen data path is entered in the Nextcloud installation wizard. Moving an existing data directory later requires a proper migration; do not move the working directory casually.

## Phase 5: Configure Apache

### Step 9 — Create the Nextcloud site configuration

Create the configuration file:

```bash
sudo nano /etc/apache2/sites-available/nextcloud.conf
```

Add:

```apache
Alias /nextcloud "/var/www/nextcloud/"

<Directory /var/www/nextcloud/>
    Require all granted
    AllowOverride All
    Options FollowSymLinks MultiViews
</Directory>
```

Save the file and enable the site and required Apache modules:

```bash
sudo a2enmod rewrite headers env dir mime setenvif
sudo a2ensite nextcloud.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

The configuration test should return `Syntax OK` before Apache is reloaded.

## Phase 6: Complete Nextcloud setup

### Step 10 — Open the installation wizard

On the Raspberry Pi, open:

```text
http://localhost/nextcloud
```

From another device on the same network, use:

```text
http://<PI-LAN-IP>/nextcloud
```

Enter the following information:

| Field | Value |
|---|---|
| Administrator username | A unique administrator name |
| Administrator password | A long, unique password |
| Data folder | The selected Nextcloud data path |
| Database type | MySQL/MariaDB |
| Database user | `nextuser` |
| Database password | The generated database password |
| Database name | `nextcloud` |
| Database host | `localhost` |

Select **Install** and wait for Nextcloud to initialize the database and create the administrator account.

### Step 11 — Verify the local deployment

Test the following:

- Log in from the Raspberry Pi browser.
- Log in from another computer or mobile device on the same network.
- Upload and download a test file.
- Create a folder and rename a file.
- Log out and log in again.
- Restart the Pi and confirm Apache, MariaDB, and Nextcloud return normally.

Useful service checks:

```bash
sudo systemctl status apache2
sudo systemctl status mariadb
sudo -u www-data php /var/www/nextcloud/occ status
```

## Phase 7: Optional external storage

### Step 12 — Prepare USB storage

List available storage and filesystem UUIDs:

```bash
lsblk -f
sudo blkid
```

Create a mount point:

```bash
sudo mkdir -p /media/nextcloud-storage
```

Use the filesystem UUID in `/etc/fstab` rather than `/dev/sda1`, because device names can change. Back up the existing file before editing it:

```bash
sudo cp /etc/fstab /etc/fstab.backup
sudo nano /etc/fstab
```

Example entry:

```text
UUID=REPLACE_WITH_FILESYSTEM_UUID /media/nextcloud-storage ext4 defaults,nofail 0 2
```

Test before rebooting:

```bash
sudo mount -a
findmnt /media/nextcloud-storage
```

Do not treat a USB drive as the only backup. Hardware failure, accidental deletion, and filesystem corruption can still destroy the data.

## Phase 8: File management

The normal and safest methods for adding files are the Nextcloud web interface, WebDAV, or an official Nextcloud client.

If an administrator copies files directly into a user's server-side data directory, Nextcloud's file cache must be updated afterward:

```bash
sudo chown -R www-data:www-data /path/to/copied/files
sudo -u www-data php /var/www/nextcloud/occ files:scan --path="USERNAME/files/Documents"
```

Replace the example username and path. Do not run a broad scan unnecessarily on a slow Raspberry Pi with a large library.

## Phase 9: Prepare for public access

The local deployment should be stable before it is exposed publicly. Complete these tasks first:

1. Update the operating system and all server components.
2. Rotate any passwords used during testing.
3. Enable Nextcloud two-factor authentication for administrators.
4. Back up the database, Nextcloud configuration, application directory, and user data.
5. Resolve warnings in Nextcloud's Administration Overview.
6. Confirm the router has no unnecessary port-forwarding rules.
7. Choose a Cloudflare-managed hostname such as `cloud.example.com`.

Follow the repository's [Cloudflare Tunnel guide](CLOUDFLARE-TUNNEL.md) to create a named tunnel, configure Nextcloud's trusted proxy settings, test the public HTTPS address, and run `cloudflared` as a system service.

Cloudflare Tunnel connects outward from the Raspberry Pi. The plan therefore does not require exposing Apache directly through the router.

## Common problems and solutions

### Nextcloud reports an untrusted domain

Add only the approved LAN address or public hostname to `trusted_domains` in Nextcloud's configuration. Do not disable the domain check.

### The browser times out

1. Confirm the Pi's current address with `hostname -I`.
2. Check that Apache is running.
3. Test from the Pi itself using `http://localhost/nextcloud`.
4. Confirm both devices are on the same local network.

### Apache does not reload

Run:

```bash
sudo apache2ctl configtest
```

Correct the reported configuration error before restarting or reloading Apache.

### Files copied directly do not appear

Fix their ownership and use `occ files:scan` for the affected user and path.

### PHP or Nextcloud reports missing requirements

Compare the installed PHP version and modules with the requirements for the selected Nextcloud release. Do not force an unsupported upgrade on the working server.

## Security and maintenance

After deployment:

- Install security updates regularly.
- Use system cron for Nextcloud background jobs.
- Review Nextcloud logs and Administration Overview warnings.
- Keep free storage space available for updates and temporary files.
- Back up the MariaDB database and user data to separate storage.
- Test backup restoration instead of assuming the backup works.
- Never commit `config.php`, tunnel credentials, database dumps, or private keys.
- Avoid exposing SSH, MariaDB, or raw storage services to the internet.

See the full [security checklist](SECURITY-CHECKLIST.md) before enabling public access.

## Project result

The completed local phase produced a working personal cloud server on a Raspberry Pi 3. Nextcloud was accessible from the Pi and other devices on the same network, and file-management operations worked through the web interface.

The next phase is to publish only the Nextcloud web service through Cloudflare Tunnel, add HTTPS and reverse-proxy settings, and test access from outside the local network. Until that phase is performed and verified, the project should be described as **locally deployed with public access planned**.

## Official references

- [Nextcloud installation documentation](https://docs.nextcloud.com/server/stable/admin_manual/installation/)
- [Nextcloud system requirements](https://docs.nextcloud.com/server/stable/admin_manual/installation/system_requirements.html)
- [Nextcloud hardening guidance](https://docs.nextcloud.com/server/stable/admin_manual/installation/harden_server.html)
- [Cloudflare Tunnel documentation](https://developers.cloudflare.com/tunnel/)
