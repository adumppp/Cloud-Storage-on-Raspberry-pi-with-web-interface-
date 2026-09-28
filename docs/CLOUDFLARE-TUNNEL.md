# Publish Nextcloud with Cloudflare Tunnel

This guide upgrades the working local Nextcloud installation to public HTTPS access without router port forwarding. Commands use placeholders—replace them deliberately and never commit generated credentials.

The examples preserve the existing `/nextcloud` web path. The resulting address is:

```text
https://cloud.example.com/nextcloud
```

## 1. Prerequisites

- A domain added to a Cloudflare account with Cloudflare nameservers active
- Working local access to `http://<PI-LAN-IP>/nextcloud`
- Current backups of the Nextcloud data directory, configuration, and MariaDB database
- Fully updated Raspberry Pi OS, Nextcloud, PHP, Apache, and MariaDB
- A unique public hostname, such as `cloud.example.com`
- Rotated database and Nextcloud administrator passwords

Do not continue if the local Nextcloud instance is already reporting database, integrity, PHP, or security errors.

## 2. Install cloudflared

Use Cloudflare's signed Debian package repository:

```bash
sudo mkdir -p --mode=0755 /usr/share/keyrings
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg >/dev/null
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
sudo apt update
sudo apt install cloudflared
cloudflared --version
```

If the repository does not support the Pi's OS release or CPU architecture, use the matching ARM/ARM64 package from Cloudflare's official download page. Do not use an unknown mirror.

## 3. Authenticate and create a named tunnel

Run these commands as the normal Raspberry Pi user, not as root:

```bash
cloudflared tunnel login
cloudflared tunnel create nextcloud-pi
cloudflared tunnel list
```

The login command opens an authorization page. Tunnel creation prints a UUID and creates a credentials JSON file under the user's `.cloudflared` directory.

Treat both `cert.pem` and the tunnel credentials JSON as secrets. Never upload them to GitHub or share them in screenshots.

## 4. Create the tunnel configuration

Create `/home/adam/.cloudflared/config.yml`, replacing all example values:

```yaml
tunnel: REPLACE_WITH_TUNNEL_UUID
credentials-file: /home/adam/.cloudflared/REPLACE_WITH_TUNNEL_UUID.json

ingress:
  - hostname: cloud.example.com
    service: http://127.0.0.1:80
  - service: http_status:404
```

If the Raspberry Pi username is not `adam`, update both paths. Restrict the directory and credential permissions:

```bash
chmod 700 /home/adam/.cloudflared
chmod 600 /home/adam/.cloudflared/*.json
```

Validate the ingress rules:

```bash
cloudflared tunnel ingress validate
cloudflared tunnel ingress rule https://cloud.example.com/nextcloud
```

## 5. Create the public DNS route

```bash
cloudflared tunnel route dns nextcloud-pi cloud.example.com
```

This creates the Cloudflare DNS record that targets the tunnel. No router port-forwarding rule is needed.

## 6. Configure Nextcloud for the reverse proxy

Back up the configuration first:

```bash
sudo cp /var/www/nextcloud/config/config.php /var/www/nextcloud/config/config.php.before-cloudflare
```

Then carefully merge the following settings into the existing `$CONFIG` array in `/var/www/nextcloud/config/config.php`:

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

Important points:

- Preserve the existing values already present in `trusted_domains`.
- Trust only the local proxy addresses used by this design; do not use `0.0.0.0/0`.
- The conditional overwrite keeps direct LAN access from being incorrectly forced to the public URL.
- If `cloudflared` runs in a container or on another machine, its proxy address will be different and must be configured precisely.

Check Nextcloud and Apache after editing:

```bash
sudo -u www-data php /var/www/nextcloud/occ config:list system
sudo apache2ctl configtest
sudo systemctl reload apache2
```

The configuration output contains sensitive values. Do not paste it publicly without redacting secrets.

## 7. Test the tunnel interactively

```bash
cloudflared tunnel run nextcloud-pi
```

While it is running, test the public address in a private browser window and from a device that is not on the home Wi-Fi:

```text
https://cloud.example.com/nextcloud
```

Confirm that login, upload, download, share links, WebDAV, and the Nextcloud mobile client work. Stop the foreground test with `Ctrl+C` only after testing.

## 8. Install it as a service

Pass the configuration path explicitly so installation with `sudo` does not look under `/root`:

```bash
sudo cloudflared --config /home/adam/.cloudflared/config.yml service install
sudo systemctl enable --now cloudflared
sudo systemctl status cloudflared
```

After a configuration change:

```bash
sudo systemctl restart cloudflared
sudo journalctl -u cloudflared --since "10 minutes ago"
```

Do not publish logs without checking them for hostnames, tunnel identifiers, local usernames, or other private information.

## 9. Security after publication

1. Enable two-factor authentication for every Nextcloud administrator.
2. Review **Administration settings → Overview** and resolve security warnings.
3. Configure Nextcloud background jobs to use system cron instead of AJAX.
4. Keep Raspberry Pi OS, Nextcloud apps, PHP, MariaDB, Apache, and cloudflared updated.
5. Maintain offline or separate-machine backups and test restoration.
6. Do not expose MariaDB, SSH, or the raw Apache port through the tunnel.
7. Keep the router's inbound port forwarding disabled.
8. Review Nextcloud logs and Cloudflare analytics for unusual activity.

Cloudflare Access can add an extra login wall for browser-only use, but it may interfere with WebDAV and Nextcloud sync clients unless those clients are planned for explicitly. Nextcloud authentication and 2FA remain required.

## 10. Rollback

If public access behaves incorrectly, stop the tunnel without changing the local Nextcloud installation:

```bash
sudo systemctl stop cloudflared
sudo systemctl disable cloudflared
```

Remove or disable the tunnel's public hostname in the Cloudflare dashboard. Local access at `http://<PI-LAN-IP>/nextcloud` should continue working. Restore the saved `config.php` only if the reverse-proxy settings themselves caused a problem.

## Official references

- [Create a locally managed Cloudflare Tunnel](https://developers.cloudflare.com/tunnel/features/locally-managed-tunnels/create-local-tunnel/)
- [Run cloudflared as a Linux service](https://developers.cloudflare.com/tunnel/features/locally-managed-tunnels/as-a-service/linux/)
- [Cloudflare Tunnel downloads](https://developers.cloudflare.com/tunnel/downloads/)
- [Nextcloud reverse-proxy configuration](https://docs.nextcloud.com/server/stable/admin_manual/configuration_server/reverse_proxy_configuration.html)
- [Nextcloud hardening guidance](https://docs.nextcloud.com/server/stable/admin_manual/installation/harden_server.html)
