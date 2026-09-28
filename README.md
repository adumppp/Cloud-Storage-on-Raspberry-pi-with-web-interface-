# Raspberry Pi Personal Cloud Storage

A self-hosted personal cloud built with **Nextcloud, Apache, PHP, and MariaDB** on a Raspberry Pi 3. The server currently works on the local network, and the next planned upgrade is secure public access through **Cloudflare Tunnel**.

## Current status

- Raspberry Pi OS 32-bit installed and updated
- Apache, PHP, and MariaDB configured
- Nextcloud installed at `/var/www/nextcloud`
- Local web access working at `http://<PI-LAN-IP>/nextcloud`
- Login tested from another device on the same network
- Cloudflare Tunnel deployment documented but not yet applied to the Pi

## Architecture

```text
Local network
    └── Browser ──> Raspberry Pi ──> Apache ──> Nextcloud ──> MariaDB/storage

Planned public access
    └── Browser ──HTTPS──> Cloudflare ──encrypted tunnel──> cloudflared on Pi
                                                            └──> Apache/Nextcloud
```

Cloudflare Tunnel uses an outbound connection from the Pi, so the planned setup does **not** require opening router ports 80 or 443.

## Documentation

- [Original local installation and troubleshooting record](docs/LOCAL-SETUP.md)
- [Cloudflare Tunnel public-access upgrade](docs/CLOUDFLARE-TUNNEL.md)
- [Security checklist](docs/SECURITY-CHECKLIST.md)

## Important security notice

The original project notes contained demonstration passwords. They have been removed from this repository. Before exposing the server publicly, rotate the Nextcloud administrator and MariaDB credentials, update the complete software stack, enable two-factor authentication, and create tested backups.

Do not commit Cloudflare tunnel credentials, Nextcloud `config.php`, database dumps, private keys, or real passwords.

## Public URL

Planned address: `https://cloud.example.com/nextcloud`

Replace `cloud.example.com` with the real hostname after the domain and Cloudflare Tunnel are configured.

## Project goal

The goal is a private alternative to hosted storage services that supports file upload, download, sharing, and synchronization while keeping the server and storage under the owner's control.

## References

- [Nextcloud Administration Manual](https://docs.nextcloud.com/server/stable/admin_manual/)
- [Cloudflare Tunnel documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Cloudflare Tunnel downloads](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/)

This repository is a deployment record and guide. It does not contain private server data, credentials, or backups.
