# Security checklist before public access

Do not enable the public Cloudflare hostname until every required item is complete.

## Required

- [ ] Replace every password copied from the original project notes
- [ ] Use different passwords for Raspberry Pi, Nextcloud admin, and MariaDB
- [ ] Confirm no password, API token, tunnel JSON, private key, or database dump is committed to Git
- [ ] Update Raspberry Pi OS and reboot if required
- [ ] Update Nextcloud and all installed apps through supported upgrade paths
- [ ] Resolve Nextcloud Administration Overview warnings
- [ ] Back up Nextcloud data, `config.php`, installed apps, and MariaDB
- [ ] Test restoring the backup on separate storage or a test machine
- [ ] Enable Nextcloud two-factor authentication for administrators
- [ ] Confirm MariaDB listens only where needed and is not exposed publicly
- [ ] Confirm the router has no inbound port forwarding for this server
- [ ] Confirm the tunnel exposes only Apache/Nextcloud
- [ ] Confirm the public URL uses HTTPS without redirect loops or mixed-content errors
- [ ] Test login, logout, upload, download, sharing, WebDAV, and mobile synchronization
- [ ] Verify the local LAN URL still works if it is meant to remain available

## Recommended improvements

- [ ] Use a router DHCP reservation for the Raspberry Pi
- [ ] Move the Nextcloud data directory outside `/var/www` during a planned migration
- [ ] Use ext4 for Linux-attached storage and mount it by filesystem UUID
- [ ] Configure Nextcloud background jobs with system cron
- [ ] Configure email delivery for security and password-reset notifications
- [ ] Maintain an offline or off-site backup
- [ ] Enable automatic security updates with a controlled reboot policy
- [ ] Monitor disk health, capacity, temperature, and backup age

## Never commit these files

- `/var/www/nextcloud/config/config.php`
- `.cloudflared/cert.pem`
- `.cloudflared/*.json`
- Database dumps
- `.env` files
- SSH private keys
- TLS private keys
- Nextcloud data or user uploads
- Screenshots containing real domains, tokens, account names, or IP information you do not want public
