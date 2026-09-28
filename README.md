# Raspberry Pi Personal Cloud Storage Server

A complete guide for deploying a self-hosted cloud-storage server with **Nextcloud, Apache, PHP, and MariaDB** on a Raspberry Pi 3.

The local-network deployment has been completed and tested. Secure public access through **Cloudflare Tunnel** is the planned second phase and is documented below without claiming that it has already been deployed.

## Current project status

- Raspberry Pi OS 32-bit installed and updated
- Apache, PHP, and MariaDB configured
- Nextcloud installed at `/var/www/nextcloud`
- Local access working at `http://<PI-LAN-IP>/nextcloud`
- Login, upload, and download tested from another local device
- Cloudflare Tunnel procedure prepared but not yet applied to the Pi

## Architecture

```text
Local deployment

Browser ──HTTP/LAN──> Raspberry Pi
                         ├── Apache
                         ├── PHP
                         ├── Nextcloud
                         ├── MariaDB
                         └── Local or USB storage

Planned public deployment

Internet browser ──HTTPS──> Cloudflare
                                │
                                └── Encrypted outbound tunnel
                                         │
                                         ▼
                                  cloudflared on Pi
                                         │
                                         ▼
                                  Apache / Nextcloud
```

Cloudflare Tunnel connects outward from the Raspberry Pi. The planned setup therefore does not require forwarding router ports 80 or 443.

## Project objectives

1. Install and configure Raspberry Pi OS.
2. Build an Apache, PHP, and MariaDB web stack.
3. Deploy Nextcloud as a private alternative to hosted storage services.
4. Access files from computers and mobile devices on the local network.
5. Prepare secure public HTTPS access through Cloudflare Tunnel.

## Requirements

### Hardware

- Raspberry Pi 3
- MicroSD card with at least 32 GB capacity
- Reliable Raspberry Pi power supply
- Monitor, keyboard, and mouse for initial setup, or SSH access
- Network connection
- Optional USB storage formatted as ext4

### Software and accounts

- Raspberry Pi OS
- Apache web server
- PHP and the required extensions
- MariaDB
- Nextcloud Server
- Modern web browser
- Cloudflare account and domain for public access

> [!WARNING]
> The original project notes contained demonstration passwords. They are intentionally excluded from this repository. Generate new passwords before following this guide and never commit Nextcloud `config.php`, Cloudflare credentials, database dumps, private keys, or user data.

## Part 1 — Local Nextcloud deployment

### Step 1 — Install Raspberry Pi OS

1. Install Raspberry Pi Imager on another computer.
2. Insert the MicroSD card.
3. Select Raspberry Pi OS and the target card.
4. Configure the username, password, Wi-Fi, locale, and SSH settings if required.
5. Write the image and insert the card into the Raspberry Pi.
6. Boot the Pi and complete the first-start setup.

The original project used Raspberry Pi OS 32-bit. For a new deployment, confirm that the chosen Raspberry Pi OS, PHP version, and Nextcloud release are compatible before continuing.

### Step 2 — Update the operating system

```bash
sudo apt update
sudo apt upgrade -y
sudo reboot
```

After rebooting, display the Pi's local address:

```bash
hostname -I
```

Configure a DHCP reservation in the router if possible so the address remains predictable.

### Step 3 — Install Apache

```bash
sudo apt install apache2 -y
sudo systemctl enable --now apache2
sudo systemctl status apache2
```

Open the following address from the Pi or another device on the same network:

```text
http://<PI-LAN-IP>/
```

The Apache default page should appear.

### Step 4 — Install PHP and its modules

```bash
sudo apt install php php-gd php-mbstring php-xml php-zip php-curl php-intl php-bz2 php-fpm php-mysql php-imagick -y
sudo systemctl restart apache2
php --version
```

Compare the displayed PHP version with the requirements of the Nextcloud version being installed.

### Step 5 — Install MariaDB

```bash
sudo apt install mariadb-server mariadb-client -y
sudo systemctl enable --now mariadb
sudo systemctl status mariadb
```

If it is available on the installed MariaDB version, run the security wizard:

```bash
sudo mariadb-secure-installation
```

On Raspberry Pi OS, the MariaDB root account may authenticate through the local Unix socket. Nextcloud should use its own restricted database account.

### Step 6 — Create the Nextcloud database

Open MariaDB:

```bash
sudo mariadb
```

Create the database and dedicated account. Replace the placeholder with a long, randomly generated password:

```sql
CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'nextuser'@'localhost' IDENTIFIED BY 'REPLACE_WITH_A_RANDOM_PASSWORD';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Test the account:

```bash
mariadb -u nextuser -p nextcloud
```

Enter the generated password. If the connection succeeds, type `EXIT;`.

### Step 7 — Download Nextcloud

The original deployment used Nextcloud's official latest-release archive:

```bash
cd /var/www
sudo wget https://download.nextcloud.com/server/releases/latest.zip
sudo apt install unzip -y
sudo unzip latest.zip
sudo chown -R www-data:www-data /var/www/nextcloud
```

Always download Nextcloud from its official domain. For a production deployment, deliberately select a supported release and verify its published checksum instead of relying permanently on an unversioned archive.

### Step 8 — Select the data directory

The completed local project used:

```text
/var/www/nextcloud/data
```

For a new public-facing installation, Nextcloud recommends storing user data outside the web root. One possible location is:

```bash
sudo mkdir -p /srv/nextcloud-data
sudo chown -R www-data:www-data /srv/nextcloud-data
sudo chmod 750 /srv/nextcloud-data
```

Enter the chosen path during the browser setup. Moving an existing Nextcloud data directory requires a proper migration; do not move a working directory casually.

### Step 9 — Configure Apache

Create the Nextcloud site configuration:

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

Enable the configuration and required modules:

```bash
sudo a2enmod rewrite headers env dir mime setenvif
sudo a2ensite nextcloud.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

The configuration test should return `Syntax OK` before Apache is reloaded.

### Step 10 — Complete the browser installation

Open this address on the Pi:

```text
http://localhost/nextcloud
```

From another device on the same network, use:

```text
http://<PI-LAN-IP>/nextcloud
```

Complete the setup wizard with these values:

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

Select **Install** and wait for Nextcloud to initialize its database.

### Step 11 — Verify the deployment

Test the following:

- Log in from the Raspberry Pi browser.
- Log in from a phone or computer on the same network.
- Upload and download a test file.
- Create a folder and rename a file.
- Log out and log in again.
- Restart the Pi and confirm that the services return.

Useful checks:

```bash
sudo systemctl status apache2
sudo systemctl status mariadb
sudo -u www-data php /var/www/nextcloud/occ status
```

## Part 2 — Optional external storage

### Step 12 — Identify and mount the USB drive

Display connected disks and filesystem UUIDs:

```bash
lsblk -f
sudo blkid
```

Create a mount point:

```bash
sudo mkdir -p /media/nextcloud-storage
```

Back up `/etc/fstab` before editing it:

```bash
sudo cp /etc/fstab /etc/fstab.backup
sudo nano /etc/fstab
```

Use a filesystem UUID rather than a temporary device name such as `/dev/sda1`:

```text
UUID=REPLACE_WITH_FILESYSTEM_UUID /media/nextcloud-storage ext4 defaults,nofail 0 2
```

Test the entry before rebooting:

```bash
sudo mount -a
findmnt /media/nextcloud-storage
```

A mounted USB drive is storage, not a backup. Keep another copy of important data on separate hardware.

### Step 13 — Import files correctly

For normal use, upload through the Nextcloud web interface, WebDAV, or an official Nextcloud client.

Files copied directly into a user's server-side data directory must be assigned to the web-server account and added to Nextcloud's file cache:

```bash
sudo chown -R www-data:www-data /path/to/copied/files
sudo -u www-data php /var/www/nextcloud/occ files:scan --path="USERNAME/files/Documents"
```

Replace the example username and path. Avoid scanning every user's entire storage unnecessarily on a slow Raspberry Pi.

## Part 3 — Public access with Cloudflare Tunnel

The following phase is planned but has not yet been completed on this project. Finish and verify the local installation before starting it.

The example public URL is:

```text
https://cloud.example.com/nextcloud
```

Replace `cloud.example.com` with a real hostname under a domain managed by Cloudflare.

### Step 14 — Complete the safety prerequisites

Before publishing the server:

1. Update Raspberry Pi OS, Nextcloud, its apps, PHP, Apache, and MariaDB.
2. Replace every password used during testing.
3. Enable Nextcloud two-factor authentication for administrators.
4. Resolve warnings under **Administration settings → Overview**.
5. Back up the database, Nextcloud configuration, installed apps, and user data.
6. Test restoring the backup.
7. Confirm the router has no unnecessary port-forwarding rules.

### Step 15 — Install cloudflared

Install it from Cloudflare's signed Debian repository:

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update
sudo apt install cloudflared
cloudflared --version
```

If the repository does not support the Pi's operating-system release or CPU architecture, use the matching ARM package from Cloudflare's official download page. Do not use an unknown mirror.

### Step 16 — Create a named tunnel

Run these commands as the normal Raspberry Pi user:

```bash
cloudflared tunnel login
cloudflared tunnel create nextcloud-pi
cloudflared tunnel list
```

The login command opens a browser authorization page. Creating the tunnel returns its UUID and creates a credentials JSON file under the user's `.cloudflared` directory.

Treat `cert.pem` and the tunnel JSON file as secrets. Never upload them to GitHub or include them in screenshots.

### Step 17 — Configure the tunnel

Create `/home/adam/.cloudflared/config.yml`, changing the username, UUID, and hostname as required:

```yaml
tunnel: REPLACE_WITH_TUNNEL_UUID
credentials-file: /home/adam/.cloudflared/REPLACE_WITH_TUNNEL_UUID.json

ingress:
  - hostname: cloud.example.com
    service: http://127.0.0.1:80
  - service: http_status:404
```

Protect the credentials:

```bash
chmod 700 /home/adam/.cloudflared
chmod 600 /home/adam/.cloudflared/*.json
```

Validate the rules:

```bash
cloudflared tunnel ingress validate
cloudflared tunnel ingress rule https://cloud.example.com/nextcloud
```

### Step 18 — Create the public DNS route

```bash
cloudflared tunnel route dns nextcloud-pi cloud.example.com
```

This creates the Cloudflare DNS record for the tunnel. It does not require opening a router port.

### Step 19 — Configure Nextcloud for the proxy

Back up the existing configuration:

```bash
sudo cp /var/www/nextcloud/config/config.php /var/www/nextcloud/config/config.php.before-cloudflare
```

Carefully merge these values into the existing `$CONFIG` array in `/var/www/nextcloud/config/config.php`:

```php
'trusted_domains' => [
    'localhost',
    'PI-LAN-IP',
    'cloud.example.com',
],
'trusted_proxies' => [
    '127.0.0.1',
    '::1',
],
'overwriteprotocol' => 'https',
'overwritewebroot' => '/nextcloud',
'overwrite.cli.url' => 'https://cloud.example.com/nextcloud',
'overwritecondaddr' => '^127\\.0\\.0\\.1$|^::1$',
```

Important rules:

- Preserve existing `trusted_domains` values.
- Trust only the local proxy addresses used by this design.
- Never use `0.0.0.0/0` as a trusted proxy.
- If `cloudflared` runs in a container or on another computer, its proxy address will be different.

Validate the configuration:

```bash
sudo -u www-data php /var/www/nextcloud/occ config:list system
sudo apache2ctl configtest
sudo systemctl reload apache2
```

The Nextcloud configuration output contains sensitive values. Redact it before sharing logs or screenshots.

### Step 20 — Test the tunnel

Run it interactively first:

```bash
cloudflared tunnel run nextcloud-pi
```

Test the public address from a private browser window and a device outside the home Wi-Fi:

```text
https://cloud.example.com/nextcloud
```

Test login, logout, upload, download, sharing, WebDAV, and the Nextcloud mobile client. Stop the foreground process with `Ctrl+C` after testing.

### Step 21 — Run cloudflared as a service

Pass the configuration path explicitly so installation with `sudo` does not search under `/root`:

```bash
sudo cloudflared --config /home/adam/.cloudflared/config.yml service install
sudo systemctl enable --now cloudflared
sudo systemctl status cloudflared
```

After changing the tunnel configuration:

```bash
sudo systemctl restart cloudflared
sudo journalctl -u cloudflared --since "10 minutes ago"
```

Check logs for tunnel identifiers, hostnames, usernames, or other private information before sharing them.

## Verification checklist

### Local server

- [ ] Apache starts automatically
- [ ] MariaDB starts automatically
- [ ] Nextcloud reports a healthy status
- [ ] Local login works
- [ ] Upload and download work
- [ ] The server returns after a reboot

### Public server

- [ ] Public hostname resolves correctly
- [ ] HTTPS works without certificate warnings
- [ ] No redirect loop occurs
- [ ] Nextcloud does not report an untrusted domain
- [ ] Login and logout work externally
- [ ] Upload and download work externally
- [ ] WebDAV and mobile synchronization work
- [ ] Local access still works if it is meant to remain enabled
- [ ] Router ports 80 and 443 are not forwarded to the Pi
- [ ] Cloudflare and Nextcloud logs show no unexpected errors

## Troubleshooting

### Untrusted domain

Add only the approved LAN address or public hostname to Nextcloud's `trusted_domains`. Never disable the trusted-domain check.

### Browser timeout

Check the Pi's address and service status:

```bash
hostname -I
sudo systemctl status apache2
sudo systemctl status mariadb
```

Test `http://localhost/nextcloud` on the Pi before troubleshooting Cloudflare.

### Apache refuses to reload

```bash
sudo apache2ctl configtest
```

Correct the reported syntax problem before reloading Apache.

### Directly copied files do not appear

Set the correct ownership and run a targeted `occ files:scan` for the affected user and path.

### Redirect loop after enabling Cloudflare

Check `trusted_proxies`, `overwriteprotocol`, `overwritewebroot`, `overwrite.cli.url`, and `overwritecondaddr`. Confirm the public URL includes `/nextcloud` if Apache still serves Nextcloud under that alias.

### Tunnel service cannot find its credentials

Check that the service was installed with the explicit configuration path and that `credentials-file` points to the correct JSON file. Confirm the cloudflared service can read it.

## Security and maintenance

- Enable two-factor authentication for administrators.
- Use system cron for Nextcloud background jobs.
- Regularly update Raspberry Pi OS, Nextcloud, its apps, PHP, Apache, MariaDB, and cloudflared.
- Review Nextcloud logs and Administration Overview warnings.
- Keep sufficient free storage for updates and temporary files.
- Back up the database, configuration, applications, and user files to separate storage.
- Test restoration instead of assuming a backup works.
- Never expose MariaDB or raw storage services through the tunnel.
- Do not expose SSH publicly unless it is separately secured and genuinely required.
- Do not commit `.cloudflared/*.json`, `cert.pem`, `config.php`, database dumps, `.env` files, or private keys.

Cloudflare Access can add another login layer for browser-only deployments, but it may interfere with WebDAV and synchronization clients unless those clients are planned for explicitly. Nextcloud authentication and two-factor authentication are still required.

## Rollback public access

If the tunnel behaves incorrectly, stop public access without removing the local server:

```bash
sudo systemctl stop cloudflared
sudo systemctl disable cloudflared
```

Remove or disable the public hostname in the Cloudflare dashboard. Local access through `http://<PI-LAN-IP>/nextcloud` should continue working. Restore the saved `config.php` only if the reverse-proxy settings caused the problem.

## Project result

The completed phase produced a working local personal cloud server on a Raspberry Pi 3. Nextcloud is accessible from the Pi and other devices on the same network, and file-management operations work through its web interface.

The next phase is to apply and verify the Cloudflare Tunnel configuration. Until external access has been tested successfully, the project status is **locally deployed with public access planned**.

## Supplemental documentation

- [Detailed Cloudflare Tunnel guide](docs/CLOUDFLARE-TUNNEL.md)
- [Local installation and troubleshooting record](docs/LOCAL-SETUP.md)
- [Security checklist](docs/SECURITY-CHECKLIST.md)
- [Standalone deployment write-up](docs/DEPLOYMENT-WRITEUP.md)

## Official references

- [Nextcloud installation documentation](https://docs.nextcloud.com/server/stable/admin_manual/installation/)
- [Nextcloud system requirements](https://docs.nextcloud.com/server/stable/admin_manual/installation/system_requirements.html)
- [Nextcloud reverse-proxy configuration](https://docs.nextcloud.com/server/stable/admin_manual/configuration_server/reverse_proxy_configuration.html)
- [Nextcloud hardening guidance](https://docs.nextcloud.com/server/stable/admin_manual/installation/harden_server.html)
- [Create a locally managed Cloudflare Tunnel](https://developers.cloudflare.com/tunnel/features/locally-managed-tunnels/create-local-tunnel/)
- [Run cloudflared as a Linux service](https://developers.cloudflare.com/tunnel/features/locally-managed-tunnels/as-a-service/linux/)

This repository contains documentation only. It intentionally excludes credentials, private configuration files, user data, and backups.
