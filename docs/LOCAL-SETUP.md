# Local Nextcloud setup record

This page documents the local Raspberry Pi deployment that was completed successfully. It has been cleaned up from the original conversation notes: real or example passwords are not preserved here.

## Hardware and software

- Raspberry Pi 3
- MicroSD card (32 GB or larger)
- Raspberry Pi OS 32-bit
- Apache web server
- PHP and required extensions
- MariaDB
- Nextcloud
- Optional ext4 USB storage

## 1. Prepare Raspberry Pi OS

Raspberry Pi Imager was used to flash Raspberry Pi OS. After the first boot, Wi-Fi and the local user were configured and the operating system was updated:

```bash
sudo apt update
sudo apt upgrade -y
```

## 2. Install the web and database stack

```bash
sudo apt install apache2 mariadb-server mariadb-client -y
sudo apt install php php-gd php-mbstring php-xml php-zip php-curl php-intl php-bz2 php-fpm php-mysql php-imagick -y
sudo systemctl restart apache2
```

The Apache test page was reachable locally and from another device on the same network.

## 3. Create the Nextcloud database

Open MariaDB:

```bash
sudo mariadb
```

Create a dedicated database and user. Replace the placeholder with a long, randomly generated password:

```sql
CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'nextuser'@'localhost' IDENTIFIED BY 'REPLACE_WITH_A_RANDOM_PASSWORD';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Do not reuse the Raspberry Pi login password, Nextcloud administrator password, or any password shown in an online tutorial.

## 4. Install Nextcloud

The project used the official Nextcloud server archive:

```bash
cd /var/www
sudo wget https://download.nextcloud.com/server/releases/latest.zip
sudo apt install unzip -y
sudo unzip latest.zip
sudo chown -R www-data:www-data /var/www/nextcloud
```

Before repeating these commands, check the current Nextcloud system requirements. A newer Nextcloud release may require a newer PHP version than the one available on an old Raspberry Pi OS installation.

## 5. Configure Apache

The local installation used `/etc/apache2/sites-available/nextcloud.conf`:

```apache
Alias /nextcloud "/var/www/nextcloud/"

<Directory /var/www/nextcloud/>
    Require all granted
    AllowOverride All
    Options FollowSymLinks MultiViews
</Directory>
```

Enable the configuration and Apache rewrite support:

```bash
sudo a2enmod rewrite
sudo a2ensite nextcloud.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

## 6. Complete the browser setup

The setup wizard was opened at:

```text
http://localhost/nextcloud
```

The wizard used:

- A unique Nextcloud administrator account
- Nextcloud data directory: `/var/www/nextcloud/data`
- Database: `nextcloud`
- Database user: `nextuser`
- Database host: `localhost`

For a new public-facing installation, Nextcloud recommends placing the data directory outside the web root, such as `/srv/nextcloud-data`. Moving an existing data directory requires a separate, carefully planned migration.

## 7. Test local access

Find the current LAN address:

```bash
hostname -I
```

From another device on the same network, open:

```text
http://<PI-LAN-IP>/nextcloud
```

Use a DHCP reservation on the router so the Pi keeps a predictable local address.

## 8. Optional USB storage

Identify the partition and filesystem UUID:

```bash
lsblk -f
sudo blkid
```

Prefer an ext4 filesystem and an `/etc/fstab` entry based on `UUID=...`, not a temporary device name such as `/dev/sda1`. Test the mount before relying on it and keep a separate backup; mounted USB storage is not itself a backup.

## 9. Adding files outside the web interface

Copying files directly into the Nextcloud data directory does not automatically add them to Nextcloud's file cache. After a controlled server-side copy, fix ownership and run a targeted scan:

```bash
sudo chown -R www-data:www-data /path/to/copied/files
sudo -u www-data php /var/www/nextcloud/occ files:scan --path="USERNAME/files/Documents"
```

Replace `USERNAME` and the destination with the real values. Prefer WebDAV, the Nextcloud desktop client, or the web interface for normal uploads.

## 10. Troubleshooting record

### Untrusted domain

Add each approved hostname or LAN address to Nextcloud's `trusted_domains`. Never disable the trusted-domain check.

### Connection timeout

Check the Pi address, Apache status, and local connectivity:

```bash
hostname -I
sudo systemctl status apache2
```

Confirm local access works before troubleshooting Cloudflare Tunnel.

### Certificate or DNS errors

A local IP address cannot receive a normal public certificate. Cloudflare Tunnel will provide HTTPS at the public Cloudflare hostname without exposing an inbound router port.

## Completed local result

The Nextcloud dashboard, mobile/LAN login, GitHub access from the Pi, and a sample Apache web page were all tested successfully. Public access is the next phase and is documented separately.
