# Traefik v3 Reverse Proxy

Docker Compose setup for Traefik v3.2 reverse proxy with automatic HTTPS via Let's Encrypt, designed for GitOps deployment with Portainer.

## Features

- **Traefik v3.2** - Latest stable version with enhanced features
- **Automatic HTTPS** - Let's Encrypt TLS certificates via TLS challenge
- **Container Discovery** - Automatic service discovery via Docker/Podman provider
- **Dashboard** - Built-in monitoring dashboard
- **Edge Agent Tunnel** - Optional port 8000 for Portainer Edge Agent WebSocket tunnels
- **Fully Parameterized** - All configuration via environment variables
- **GitOps Ready** - No secrets in repository
- **Portainer Compatible** - Easy deployment and updates

## Prerequisites

- Docker or Podman runtime
- Portainer (for GitOps deployment)
- Domain names pointing to your server
- Ports 80 and 443 available
- Port 8000 available (optional, only if using Portainer Edge Agent tunnel)

## Migration from Traefik v2

If you're upgrading from Traefik v2.10, this repository provides a seamless migration path while maintaining all existing functionality.

### Pre-Migration Checklist

1. **Backup ACME Certificates**:
   ```bash
   ssh your-server.com
   sudo podman volume inspect traefik_traefik-certs
   # Note the Mountpoint path
   sudo tar -czf ~/traefik-certs-backup-$(date +%Y%m%d).tar.gz /var/lib/containers/storage/volumes/traefik_traefik-certs/_data
   ```

2. **Document Current State**:
   ```bash
   sudo podman ps --filter name=traefik
   sudo podman inspect traefik-traefik-1 > ~/traefik-v2-config.json
   ```

3. **List Current Services**:
   ```bash
   sudo podman ps --format "{{.Names}}" | grep -v traefik
   # Note all services that depend on Traefik
   ```

### Migration Steps

#### Option 1: Direct Upgrade (Recommended)

This approach stops the old Traefik v2 container and deploys Traefik v3 using the same network and ACME volume.

**Downtime**: ~1-2 minutes

1. **Deploy Stack in Portainer** (DO NOT start yet):
   - Go to **Stacks** → **Add Stack**
   - **Name**: `traefik`
   - **Build method**: **Repository**
   - **Repository URL**: `https://github.com/yourusername/traefik-compose`
   - **Repository reference**: `main`
   - **Compose path**: `docker-compose.yml`
   - Configure environment variables (see below)
   - **DO NOT** click "Deploy" yet

2. **Configure Environment Variables**:
   ```
   TRAEFIK_IMAGE=traefik:v3.2
   CONTAINER_NAME=traefik
   NETWORK_NAME=traefik_default
   ACME_EMAIL=your-email@example.com
   CERT_RESOLVER_NAME=myresolver
   CERT_VOLUME_NAME=traefik_traefik-certs
   DASHBOARD_HOST=traefik.yourdomain.com
   LOG_LEVEL=INFO
   ```

3. **Stop Old Traefik v2**:
   ```bash
   ssh your-server.com
   sudo podman stop traefik-traefik-1
   # DO NOT remove - keep as rollback option
   ```

4. **Deploy New Traefik v3 Stack** in Portainer:
   - Click **Deploy the stack**
   - Monitor the logs for successful startup

5. **Verify Services**:
   ```bash
   # Check Traefik is running
   sudo podman ps --filter name=traefik

   # Check logs for errors
   sudo podman logs traefik

   # Verify dashboard access
   curl -I https://traefik.yourdomain.com

   # Test a protected service
   curl -I https://app.yourdomain.com
   ```

6. **Test All Services**:
   - Access each of your ~20 services
   - Verify HTTPS certificates are working
   - Check oauth2-proxy protected services

7. **Remove Old Container** (after confirming everything works):
   ```bash
   sudo podman rm traefik-traefik-1
   ```

#### Option 2: Rollback Procedure

If something goes wrong during migration:

1. **Stop Traefik v3**:
   - In Portainer: Stop the traefik stack

2. **Start Traefik v2**:
   ```bash
   ssh your-server.com
   sudo podman start traefik-traefik-1
   ```

3. **Verify**:
   ```bash
   sudo podman ps --filter name=traefik
   sudo podman logs traefik-traefik-1
   ```

4. **Debug Traefik v3**:
   - Check logs in Portainer
   - Review environment variables
   - Verify network and volume names match

### Post-Migration: OAuth2-Proxy Enhancement

After successfully migrating to Traefik v3, you can enhance the oauth2-proxy setup to automatically redirect unauthenticated users to the sign-in page.

**This feature was not possible in Traefik v2!**

To enable automatic redirects, update your protected services (e.g., vaultwarden) with error middleware:

```yaml
labels:
  - "traefik.http.routers.vaultwarden.middlewares=forward-auth@docker,auth-errors"
  - "traefik.http.middlewares.auth-errors.errors.status=401-403"
  - "traefik.http.middlewares.auth-errors.errors.service=oauth@docker"
  - "traefik.http.middlewares.auth-errors.errors.query=/oauth2/start?rd={url}"
```

This configuration:
1. Checks authentication via `forward-auth` middleware
2. If user is not authenticated (401/403), redirects to oauth2-proxy sign-in
3. After successful authentication, redirects back to original URL

## Quick Start with Portainer

### 1. Deploy in Portainer

1. Go to **Stacks** → **Add Stack**
2. **Name**: `traefik`
3. **Build method**: Select **Repository**
4. **Repository URL**: `https://github.com/yourusername/traefik-compose`
5. **Repository reference**: `main`
6. **Compose path**: `docker-compose.yml`

### 2. Configure Environment Variables

**Required variables:**

```
TRAEFIK_IMAGE=traefik:v3.2
CONTAINER_NAME=traefik
DOCKER_SOCKET=/var/run/docker.sock
NETWORK_NAME=traefik_default
ACME_EMAIL=your-email@example.com
CERT_RESOLVER_NAME=myresolver
CERT_VOLUME_NAME=traefik_traefik-certs
DASHBOARD_HOST=traefik.yourdomain.com
```

**Important:** For Podman hosts, set:
```
DOCKER_SOCKET=/var/run/podman/podman.sock
```

**Optional variables** (use defaults if not specified):

```
HTTP_PORT=80
HTTPS_PORT=443
TRAEFIK_DASHBOARD=true
LOG_LEVEL=INFO
```

**For Portainer Edge Agent support**, uncomment the port 8000 lines in `docker-compose.yml` and set:
```
EDGE_TUNNEL_PORT=8000
```

### 3. Deploy

Click **Deploy the stack**

### 4. **CRITICAL: Connect to Shared Network** (If Using OAuth2-Proxy or Other Services)

If you're using oauth2-proxy or other services that need to communicate with Traefik (via forward-auth middleware, for example), you **must** manually connect Traefik to the shared network after deployment:

```bash
ssh your-server.com
sudo podman network connect traefik_default traefik
```

**Why is this needed?**

When Traefik starts, it joins its default Podman network. However, oauth2-proxy and other services typically run on the `traefik_default` network. Without connecting Traefik to this shared network:
- The `forward-auth` middleware cannot reach oauth2-proxy
- All OAuth-protected services will return **500 Internal Server Error**
- Traefik won't be able to route to services on the shared network

**Verify network connectivity:**

```bash
# Check Traefik is on both networks (should show: podman and traefik_default)
sudo podman inspect traefik --format '{{range $name, $network := .NetworkSettings.Networks}}{{printf "%s\n" $name}}{{end}}'

# Test connectivity to oauth2-proxy (if applicable)
sudo podman exec traefik ping -c 2 oauth2-proxy
```

### 5. Verify

Check the dashboard at `https://traefik.yourdomain.com`

## Configuration Reference

### Required Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `ACME_EMAIL` | Email for Let's Encrypt notifications | `admin@example.com` |
| `DASHBOARD_HOST` | Hostname for Traefik dashboard | `traefik.yourdomain.com` |
| `DOCKER_SOCKET` | Docker/Podman socket path | `/var/run/docker.sock` (Docker) or `/var/run/podman/podman.sock` (Podman) |
| `NETWORK_NAME` | Docker network name (shared with other services) | `traefik_default` |

### Optional Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `TRAEFIK_IMAGE` | Traefik Docker image | `traefik:v3.2` |
| `CONTAINER_NAME` | Container name | `traefik` |
| `HTTP_PORT` | HTTP port | `80` |
| `HTTPS_PORT` | HTTPS port | `443` |
| `EDGE_TUNNEL_PORT` | Portainer Edge Agent tunnel port (TCP/WebSocket) | `8000` |
| `CERT_RESOLVER_NAME` | Certificate resolver name | `myresolver` |
| `CERT_VOLUME_NAME` | Volume name for ACME storage | `traefik_traefik-certs` |
| `TRAEFIK_DASHBOARD` | Enable dashboard | `true` |
| `LOG_LEVEL` | Logging level (DEBUG, INFO, WARN, ERROR) | `INFO` |

## Using Traefik with Other Services

### Basic Service Configuration

Add these labels to any service you want to expose via Traefik:

```yaml
services:
  myapp:
    image: myapp:latest
    networks:
      - traefik_default
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`myapp.yourdomain.com`)"
      - "traefik.http.routers.myapp.entrypoints=websecure"
      - "traefik.http.routers.myapp.tls.certresolver=myresolver"
      - "traefik.http.services.myapp.loadbalancer.server.port=8080"

networks:
  traefik_default:
    external: true
```

### Protected Service with OAuth2-Proxy

For services that require authentication via oauth2-proxy:

```yaml
services:
  myapp:
    image: myapp:latest
    networks:
      - traefik_default
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`myapp.yourdomain.com`)"
      - "traefik.http.routers.myapp.entrypoints=websecure"
      - "traefik.http.routers.myapp.tls.certresolver=myresolver"
      - "traefik.http.services.myapp.loadbalancer.server.port=8080"

      # Forward authentication
      - "traefik.http.routers.myapp.middlewares=forward-auth@docker,auth-errors"

      # Automatic redirect to sign-in (Traefik v3 feature!)
      - "traefik.http.middlewares.auth-errors.errors.status=401-403"
      - "traefik.http.middlewares.auth-errors.errors.service=oauth@docker"
      - "traefik.http.middlewares.auth-errors.errors.query=/oauth2/start?rd={url}"

networks:
  traefik_default:
    external: true
```

## Troubleshooting

### Check Logs

```bash
# Via Portainer: Logs tab in the stack

# Via SSH:
sudo podman logs traefik
```

### Common Issues

1. **"Cannot connect to the Docker daemon" error in logs**:
   - **Cause**: Wrong Docker/Podman socket path
   - **Solution**: Set the correct socket path in Portainer environment variables:
     - For Docker: `DOCKER_SOCKET=/var/run/docker.sock`
     - For Podman: `DOCKER_SOCKET=/var/run/podman/podman.sock`
   - **Verify** which runtime you're using:
     ```bash
     docker --version  # If this works, use /var/run/docker.sock
     podman --version  # If this works, use /var/run/podman/podman.sock
     ```

2. **500 Internal Server Error on OAuth-protected sites**:
   - **Cause**: Traefik cannot reach oauth2-proxy (not on same network)
   - **Solution**: Connect Traefik to the shared network:
     ```bash
     sudo podman network connect traefik_default traefik
     ```
   - **Verify**:
     ```bash
     sudo podman exec traefik ping -c 2 oauth2-proxy
     ```

3. **Port already in use**:
   - Check if old Traefik v2 is still running: `sudo podman ps --filter name=traefik`
   - Stop it: `sudo podman stop traefik-traefik-1`

4. **ACME certificates not working**:
   - Verify email is set: Check `ACME_EMAIL` variable
   - Check volume name matches: `CERT_VOLUME_NAME=traefik_traefik-certs`
   - Verify port 443 is accessible from internet

5. **Dashboard not accessible**:
   - Check `DASHBOARD_HOST` is correct
   - Verify DNS points to your server
   - Check dashboard is enabled: `TRAEFIK_DASHBOARD=true`

6. **Services not discovered**:
   - Verify services are on same network: `NETWORK_NAME=traefik_default`
   - Check service has `traefik.enable=true` label
   - Verify socket path is correct: `DOCKER_SOCKET`

7. **Edge Agent tunnel connection refused (port 8000)**:
   - Verify port 8000 is exposed: `sudo podman port traefik | grep 8000`
   - Check firewall allows TCP on port 8000
   - Verify edge-tunnel entrypoint is configured

8. **Migration failed**:
   - Use rollback procedure (see above)
   - Check logs for errors: `sudo podman logs traefik`
   - Verify all environment variables are set correctly

## What's New in Traefik v3

Traefik v3 brings several improvements over v2:

- **Enhanced Middleware**: Better error handling with dynamic placeholders
- **Improved Performance**: Faster routing and service discovery
- **Better Observability**: Enhanced metrics and logging
- **HTTP/3 Support**: Ready for QUIC protocol
- **Kubernetes Gateway API**: Native support for Gateway API

### Breaking Changes from v2

Traefik v3 maintains backward compatibility for most features. However, be aware:

- File provider syntax updates (not used in this setup)
- Some deprecated v2 options removed (this setup uses current syntax)
- Middleware priorities changed (thoroughly tested in this setup)

## Security Notes

- Never commit `.env` files with real secrets
- Restrict dashboard access with basic auth or IP whitelist
- Regularly update to latest Traefik v3 patch releases
- Monitor logs for unusual activity
- Use strong ACME email (for certificate expiry notifications)

## Updates and Maintenance

### Updating Traefik

In Portainer:
1. Go to **Stacks** → **traefik**
2. Change `TRAEFIK_IMAGE` to new version (e.g., `traefik:v3.3`)
3. Click **Update the stack**

### Viewing Certificate Status

```bash
ssh your-server.com
sudo podman exec traefik cat /etc/traefik/acme/acme.json
```

## License

MIT

## Resources

- [Traefik v3 Documentation](https://doc.traefik.io/traefik/)
- [Traefik v3 Migration Guide](https://doc.traefik.io/traefik/migration/v2-to-v3/)
- [Traefik Docker Provider](https://doc.traefik.io/traefik/providers/docker/)
- [Let's Encrypt](https://letsencrypt.org/)
