# Docker Registry with Traefik

A production-ready private Docker Registry with Traefik reverse proxy, automatic HTTPS via Let's Encrypt, and basic authentication.

## Features

- **Traefik** reverse proxy with automatic HTTP to HTTPS redirect
- **Let's Encrypt** SSL certificates (auto-renewal)
- **Basic authentication** for registry access
- **Flexible configuration** via environment variables

## Prerequisites

- Docker and Docker Compose installed
- A domain name pointing to your server
- Ports 80 and 443 available
- `htpasswd` utility (usually part of `apache2-utils` package)

## Quick Start

### 1. Clone and configure

```bash
# Clone the repository
git clone <repository-url>
cd docker-registry-with-traefik

# Create environment file
cp .example.env .env
```

### 2. Edit the `.env` file

```bash
# Required: Set your domain
DOMAIN=registry.yourdomain.com

# Required: Set your email for Let's Encrypt
LETSENCRYPT_EMAIL=your-email@example.com

# Required: Generate a secret key
REGISTRY_HTTP_SECRET=$(openssl rand -hex 32)
```

### 3. Create authentication file

```bash
# Create the first user
htpasswd -Bbn username password > .htpasswd
```

### 4. Start the services

```bash
docker compose up -d
```

Your registry is now available at `https://your-domain.com`

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `DOMAIN` | Your domain name | `example.com` |
| `TRAEFIK_IMAGE` | Traefik Docker image | `traefik:3.6.4` |
| `REGISTRY_IMAGE` | Registry Docker image | `registry:2` |
| `LETSENCRYPT_EMAIL` | Email for SSL notifications | `admin@example.com` |
| `LETSENCRYPT_CA_SERVER` | ACME server URL | Production server |
| `REGISTRY_HTTP_SECRET` | Secret key for registry | - |

### Staging SSL Certificates

For testing, use Let's Encrypt staging server to avoid rate limits:

```bash
LETSENCRYPT_CA_SERVER=https://acme-staging-v02.api.letsencrypt.org/directory
```

## User Management

### Create a new user

```bash
# Create .htpasswd file with first user
htpasswd -Bbn username password > .htpasswd

# Or interactively (prompts for password)
htpasswd -B -c .htpasswd username
```

### Add additional users

```bash
# Append a new user to existing file
htpasswd -Bbn newuser newpassword >> .htpasswd

# Or interactively
htpasswd -B .htpasswd newuser
```

### Remove a user

```bash
htpasswd -D .htpasswd username
```

### List users

```bash
cat .htpasswd | cut -d: -f1
```

### Update user password

```bash
htpasswd -B .htpasswd existinguser
```

> **Note:** After modifying the `.htpasswd` file, restart Traefik to apply changes:
> ```bash
> docker compose restart traefik
> ```

## Usage

### Login to registry

```bash
docker login your-domain.com
# Enter username and password when prompted
```

### Push an image

```bash
# Tag your image
docker tag myimage:latest your-domain.com/myimage:latest

# Push to registry
docker push your-domain.com/myimage:latest
```

### Pull an image

```bash
docker pull your-domain.com/myimage:latest
```

### List repositories

```bash
curl -u username:password https://your-domain.com/v2/_catalog
```

### List image tags

```bash
curl -u username:password https://your-domain.com/v2/myimage/tags/list
```

## Maintenance

### View logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f traefik
docker compose logs -f registry
```

### Restart services

```bash
docker compose restart
```

### Stop services

```bash
docker compose down
```

### Backup registry data

```bash
# Create backup
docker run --rm -v traefik-docker-registry_registry_data:/data -v $(pwd):/backup alpine tar cvzf /backup/registry-backup.tar.gz /data

# Restore backup
docker run --rm -v traefik-docker-registry_registry_data:/data -v $(pwd):/backup alpine sh -c "cd /data && tar xvzf /backup/registry-backup.tar.gz --strip 1"
```

### Garbage collection

Remove unused layers to free up disk space:

```bash
docker compose exec registry bin/registry garbage-collect /etc/docker/registry/config.yml
```

## Troubleshooting

### Certificate issues

1. Check Traefik logs: `docker compose logs traefik`
2. Ensure ports 80/443 are accessible from the internet
3. Verify DNS is pointing to your server
4. Try staging CA server first to avoid rate limits

### Authentication issues

1. Verify `.htpasswd` file exists and has correct format
2. Check file permissions: `chmod 644 .htpasswd`
3. Restart Traefik after `.htpasswd` changes

### Registry health check

```bash
curl -I https://your-domain.com/v2/
# Should return 401 Unauthorized (auth required)

curl -u username:password https://your-domain.com/v2/
# Should return {} (empty JSON)
```

### Rootless Docker setup

If you're running Docker in rootless mode, you may encounter errors binding to ports 80 and 443 (privileged ports). To fix this:

1. Edit `/etc/sysctl.conf`:

```bash
sudo nano /etc/sysctl.conf
```

2. Add the following line:

```
net.ipv4.ip_unprivileged_port_start=0
```

3. Apply the changes:

```bash
sudo sysctl -p
```

4. Restart Docker and your containers:

```bash
systemctl --user restart docker
docker compose up -d
```

> **Note:** This allows unprivileged users to bind to all ports (0+). If you prefer a more restrictive setting, use `net.ipv4.ip_unprivileged_port_start=80` to only allow ports 80 and above.

## License

MIT
