# Traefik Reverse Proxy for NAS

This Docker Compose setup configures Traefik as a reverse proxy with automatic Let's Encrypt SSL certificates using DNS challenge via DigitalOcean.

## Features

- Automatic HTTPS with Let's Encrypt SSL certificates
- DNS challenge using DigitalOcean (works for internal IPs)
- Automatic certificate renewal
- HTTP to HTTPS redirection
- Traefik dashboard for monitoring
- Easy to extend with additional services

## Prerequisites

1. Docker and Docker Compose installed on your NAS
2. Domain `console.lol` with DNS pointing to your NAS IP (10.0.0.54)
3. DigitalOcean account with API token for DNS challenge
4. DNS records configured:
   - `*.console.lol` → 10.0.0.54 (wildcard A record)
   - Or individual A records for each subdomain

## Setup Instructions

### 1. Configure DNS on DigitalOcean

In your DigitalOcean DNS panel for `console.lol`:

```
Type: A
Hostname: *
Value: 10.0.0.54
TTL: 3600
```

Or create individual A records:

- `nas.console.lol` → 10.0.0.54
- `syncthing.console.lol` → 10.0.0.54
- `traefik.console.lol` → 10.0.0.54

### 2. Get DigitalOcean API Token

1. Go to https://cloud.digitalocean.com/account/api/tokens
2. Generate a new token with read/write access
3. Copy the token

### 3. Configure Environment Variables

```bash
cp .env.example .env
```

Edit `.env` and add your DigitalOcean API token:

```
DO_AUTH_TOKEN=your_actual_token_here
```

### 4. Update Email in traefik.yml

Edit `traefik/traefik.yml` and replace `your-email@example.com` with your actual email address for Let's Encrypt notifications.

### 5. Create Docker Network

```bash
docker network create proxy
```

### 6. Start Traefik

```bash
docker compose up -d
```

### 7. Check Logs

```bash
docker compose logs -f traefik
```

Watch for successful certificate generation. It should show:

- Certificate obtained for `console.lol` and `*.console.lol`

## Configured Services

The following services are pre-configured:

| Service           | URL                           | Backend                |
| ----------------- | ----------------------------- | ---------------------- |
| NAS ADM           | https://nas.console.lol       | http://10.0.0.54:8001  |
| Syncthing         | https://syncthing.console.lol | http://10.0.0.54:28384 |
| Traefik Dashboard | https://traefik.console.lol   | Internal               |
| RustFS S3 API     | https://s3.console.lol        | http://localhost:9000  |
| RustFS Console    | https://rustfs.console.lol    | http://localhost:9001  |

## RustFS

RustFS runs as a separate app in `apps/rustfs` and is routed by the existing Traefik file provider.

RustFS data is stored in the external Docker volume `rustfs_rustfs_data_v2`. Portainer stack updates reuse this volume and cannot remove it as part of the Compose project lifecycle. The volume must exist before the first deployment; create it in Portainer or with:

```bash
docker volume create rustfs_rustfs_data_v2
```

Store local RustFS credentials in the macOS keychain through Fnox:

```bash
fnox set --provider keychain RUSTFS_ACCESS_KEY
fnox set --provider keychain RUSTFS_SECRET_KEY
```

Run it locally with mise:

```bash
mise run rustfs:config
mise run rustfs:up
mise run rustfs:health
```

The local console URL is `http://localhost:9001/rustfs/console/`. The Traefik route redirects `https://rustfs.console.lol` to that console path automatically and routes non-console RustFS requests on that host to the S3/API port so the console can use the current host as its default server address.

For Portainer GitOps, set `RUSTFS_ACCESS_KEY` and `RUSTFS_SECRET_KEY` as stack environment variables rather than committing an `.env` file.

Also set `RUSTFS_IAM_MASTER_KEY` to a new, strong generated secret and keep it stable. When recovering IAM data encrypted with previous root credentials, temporarily set `RUSTFS_IAM_MASTER_KEY_OLD_KEYS` to the old secret key and the old access/secret pair as comma-separated candidates:

```text
<old-secret-key>,<old-access-key>:<old-secret-key>
```

After RustFS has loaded IAM successfully and rewritten the legacy records, remove `RUSTFS_IAM_MASTER_KEY_OLD_KEYS`. Do not remove or rotate `RUSTFS_IAM_MASTER_KEY` without another key-rotation migration.

### Traefik Dashboard Access

The dashboard is protected with basic auth:

- Username: `admin`
- Password: `admin` (change this!)

To generate a new password hash:

```bash
echo $(htpasswd -nb admin your_password) | sed -e s/\\$/\\$\\$/g
```

Update the hash in `docker-compose.yml` under the `traefik-auth.basicauth.users` label.

## Adding New Services

To add a new service, edit `traefik/config.yml`:

```yaml
http:
  routers:
    your-service-name:
      entryPoints:
        - 'https'
      rule: 'Host(`your-service.console.lol`)'
      middlewares:
        - default-headers
      tls:
        certResolver: digitalocean
      service: your-service-name

  services:
    your-service-name:
      loadBalancer:
        servers:
          - url: 'http://10.0.0.54:PORT'
        passHostHeader: true
```

Then reload Traefik:

```bash
docker compose restart traefik
```

## File Structure

```
.
├── docker-compose.yml          # Main Docker Compose configuration
├── .env                        # Environment variables (DO NOT commit)
├── .env.example               # Example environment file
├── traefik/
│   ├── traefik.yml           # Static Traefik configuration
│   ├── config.yml            # Dynamic configuration (routes/services)
│   └── acme.json             # Let's Encrypt certificates storage
└── README.md                 # This file
```

## Troubleshooting

### Certificates not generating

1. Check DigitalOcean API token is correct
2. Verify DNS records are propagated: `dig nas.console.lol`
3. Check Traefik logs: `docker compose logs -f traefik`
4. Ensure `acme.json` has 600 permissions

### Service not accessible

1. Check the backend service is running on the specified port
2. Verify firewall allows ports 80 and 443
3. Check Traefik dashboard for router status
4. Test backend directly: `curl http://10.0.0.54:PORT`

### Certificate renewal issues

Certificates auto-renew 30 days before expiration. Check logs around renewal time.

## Security Notes

- The `.env` file contains sensitive API tokens - never commit it to version control
- Change the default Traefik dashboard password
- Consider restricting access to the Traefik dashboard to local network only
- The `default-whitelist` middleware is configured for private IP ranges

## Updating Configuration

After making changes to `traefik/config.yml`:

```bash
docker compose restart traefik
```

After making changes to `traefik/traefik.yml` or `docker-compose.yml`:

```bash
docker compose down
docker compose up -d
```

## Stopping Traefik

```bash
docker compose down
```

## Resources

- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Let's Encrypt DNS Challenge](https://doc.traefik.io/traefik/https/acme/#dnschallenge)
- [DigitalOcean DNS Provider](https://go-acme.github.io/lego/dns/digitalocean/)
