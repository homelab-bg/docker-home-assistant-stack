# Home Assistant Stack

Home Assistant with ESPHome, Matter Server and the Everything Presence mmWave Configurator. Optional overlays add Traefik reverse proxying (TLS via Let's Encrypt), a Mosquitto MQTT broker, and a Cloudflare Tunnel.

| File | Adds |
|---|---|
| `docker-compose.yml` | Base: Home Assistant, Matter Server |
| `docker-compose.traefik.yml` | Traefik routing + TLS for Home Assistant |
| `docker-compose.esp.yml` | ESPHome, Everything Presence mmWave Configurator |
| `docker-compose.esp.traefik.yml` | Traefik routing + TLS for ESPHome and EP configurator - needs both `esp` and `traefik` overlays |
| `docker-compose.mqtt.yml` | Mosquitto on 1883 (LAN-only, no TLS) - skip if another broker (e.g. an SMHub) already handles MQTT |
| `docker-compose.cloudflared.yml` | Cloudflare Tunnel for external access |

Home Assistant, ESPHome and Matter Server use `network_mode: host` - needed for mDNS/SSDP discovery, Matter and Bluetooth - so they can't join a Docker network. The Traefik overlays route to them via the host instead (see [Traefik Integration](#traefik-integration)).

## Quick Start

1. Copy environment template:
```bash
cp .env.example .env
```

2. Edit `.env` - set the `*_HOST` values if using Traefik. With the ESP overlay, the EP configurator reaches HA directly via `host.docker.internal:8123` by default, so `HA_BASE_URL` only needs setting for an HA elsewhere.

3. ESP overlay only: create the EP configurator's token file (gitignored). The token is a long-lived token from HA (**Profile → Security**), so this step can wait until HA is running:
```bash
mkdir -p secrets
echo -n "your-long-lived-token" > secrets/ha_long_lived_token
```

4. Set data directory ownership - see [Permissions](#permissions).

## Deployment Options

Combine the base with whichever overlays you need, e.g.:

```bash
# Standalone
docker compose up -d

# Traefik
docker compose -f docker-compose.yml -f docker-compose.traefik.yml up -d

# Traefik + ESP
docker compose -f docker-compose.yml -f docker-compose.traefik.yml \
  -f docker-compose.esp.yml -f docker-compose.esp.traefik.yml up -d

# Everything
docker compose -f docker-compose.yml -f docker-compose.traefik.yml \
  -f docker-compose.esp.yml -f docker-compose.esp.traefik.yml \
  -f docker-compose.mqtt.yml -f docker-compose.cloudflared.yml up -d
```

Standalone access: Home Assistant at `http://<host>:8123`; with the ESP overlay, ESPHome at `http://<host>:6052` and EP configurator at `http://<host>:42069`.

### Traefik Integration

Requires a Traefik instance on the same host with Let's Encrypt configured (e.g. [docker-traefik-portainer](https://github.com/homelab-bg/docker-traefik-portainer)), and an external `traefik` network.

**Traefik must be able to reach host-network containers.** For those, Traefik's Docker provider forwards to `host.docker.internal`, falling back to `127.0.0.1` (i.e. Traefik's own container). On Linux that only resolves if the Traefik container has:
```yaml
extra_hosts:
  - host.docker.internal:host-gateway
```
docker-traefik-portainer sets this already.

**Home Assistant must trust Traefik**, or it rejects proxied requests with `400: Bad Request` and logs `your HTTP integration is not set-up for reverse proxies`. Find Traefik's subnets - dual-stack networks print an IPv6 one too; trust it as well, or proxied IPv6 requests get rejected:
```bash
docker network inspect traefik --format '{{range .IPAM.Config}}{{println .Subnet}}{{end}}'
```
Don't assume Docker's default `172.16.0.0/12` range - it won't match hosts with a custom Docker address pool (e.g. `172.32.0.0/16`).

- **HA 2026.8+**: HTTP settings are configured in the UI - an `http:` block in `configuration.yaml` is ignored (it's imported once, on the first start after upgrading, so a fresh install never picks it up). Browse to HA directly at `http://<host-ip>:8123` (host networking bypasses Traefik), then **Settings → System → Network → HTTP server**: turn on **Trust X-Forwarded-For** and add each subnet above to **Trusted proxies**.

  Alternatively, re-trigger the one-time YAML import: HA stores these settings, plus a "migration done" flag, in `.storage/http`. With the `http:` block below in `configuration.yaml`, stop HA, delete that file and start it again - the block is imported on that start, nothing else is reset:
  ```bash
  docker stop home-assistant
  rm ha/config/.storage/http
  docker start home-assistant
  ```
  Then remove the `http:` block - it's ignored from then on, and raises a deprecation repair issue while present (YAML support is removed in 2027.2.0).
- **Before 2026.8** (or for the import above): add to `ha/config/configuration.yaml`:
  ```yaml
  http:
    use_x_forwarded_for: true
    trusted_proxies:
      - 172.18.0.0/16   # replace with the subnets above
      - fd00::/64       # IPv6 subnet, if the network has one
  ```

### MQTT

LAN-only IoT on 1883, no TLS. Configure auth before first start - `mqtt/config/mosquitto.conf`:
```
listener 1883
allow_anonymous false
password_file /mosquitto/config/passwd
persistence true
persistence_location /mosquitto/data/
log_dest file /mosquitto/log/mosquitto.log
```
Create the password file:
```bash
docker compose -f docker-compose.yml -f docker-compose.mqtt.yml run --rm mqtt \
  mosquitto_passwd -c /mosquitto/config/passwd <username>
```

### Cloudflare Tunnel

The tunnel token is read from a secrets file (gitignored), not an env var - plain env vars are visible via `docker inspect`:
```bash
mkdir -p secrets
echo -n "your-tunnel-token" > secrets/cloudflare_tunnel_token
```
Requires cloudflared 2025.4.0+ (`TUNNEL_TOKEN_FILE` support).

## Permissions

None of these images read `PUID`/`PGID`, and several run as a fixed non-root user. Docker creates missing bind-mount directories as root, so set their ownership once before first start:

| Container | Runs as | Needs |
|---|---|---|
| matter-server | `1000:1000` | Write access to `matter-server/data` - otherwise `EACCES: permission denied, mkdir '/data/config'` |
| mqtt | `1883:1883` (entrypoint chowns `data` only) | Write access to `mqtt/log` if logging to file |
| cloudflared | `65532:65532` | Read access to `secrets/cloudflare_tunnel_token` |
| home-assistant, esphome, ep-configurator | root | - |

```bash
mkdir -p matter-server/data mqtt/{config,data,log}
chown -R 1000:1000 matter-server/data
chown -R 1883:1883 mqtt/data mqtt/log
chmod 644 secrets/cloudflare_tunnel_token   # or: chown 65532 + chmod 400
```

Files Home Assistant and ESPHome create are owned by root, so editing them from another container (e.g. code-server running as a non-root user) needs matching permissions.

## mDNS (port 5353)

Home Assistant's zeroconf (plus ESPHome and Matter) share UDP 5353 with the host. If something on the host holds it exclusively, HA logs `OSError: [Errno 98] Address in use` for `('', 5353)`, and `zeroconf`, `ssdp`, `cloud` and `default_config` fail to set up. Find the owner with:
```bash
ss -ulpn 'sport = :5353'
```
- **TrueNAS**: TrueNAS's `avahi-daemon` (its mDNS service announcement) binds 5353 exclusively. Disable it under **Network → Global Configuration → Service Announcement → mDNS**, then restart the app. TrueNAS then stops advertising itself via mDNS (`<hostname>.local`, Mac SMB/Time Machine discovery). TrueNAS generates avahi's config, so don't edit `avahi-daemon.conf` directly.
- **VM with avahi**: set `disallow-other-stacks=no` in `/etc/avahi/avahi-daemon.conf` and restart avahi, or disable avahi.

## TrueNAS Deployment

Put this repo in a dataset (e.g. `/mnt/<pool>/Apps/home-assistant-stack`), with Traefik bound to a secondary IP via docker-traefik-portainer's TrueNAS overlay. Then **Apps → Discover Apps → Install via YAML**, listing the base plus whichever overlays you want:
```yaml
include:
  - path:
      - /mnt/<pool>/Apps/home-assistant-stack/docker-compose.yml
      - /mnt/<pool>/Apps/home-assistant-stack/docker-compose.traefik.yml
      #- /mnt/<pool>/Apps/home-assistant-stack/docker-compose.esp.yml
      #- /mnt/<pool>/Apps/home-assistant-stack/docker-compose.esp.traefik.yml
      #- /mnt/<pool>/Apps/home-assistant-stack/docker-compose.mqtt.yml
      #- /mnt/<pool>/Apps/home-assistant-stack/docker-compose.cloudflared.yml

# Web Portal buttons in the TrueNAS Apps UI. These go here, not in the compose
# files, because `include:` drops top-level x- extensions from included files.
# Use literal hostnames: .env isn't applied to this file.
x-portals:
  - {name: Home Assistant, scheme: https, host: ha.yourdomain.com, port: 443, path: /}
  #- {name: ESPHome, scheme: https, host: esphome.yourdomain.com, port: 443, path: /}   # with the ESP overlays
```

The dataset is the project directory, so `.env`, `./secrets` and the app data directories all resolve inside it. Set `MATTER_INTERFACE` in `.env` to the TrueNAS NIC carrying your LAN IP (`ip -br addr`) - TrueNAS doesn't use `eth0`-style names, and a name that doesn't exist on the host crashes matter-server with `Unknown interface`.

## Security

⚠️ Host networking means Home Assistant (8123) and ESPHome (6052) stay reachable over plain HTTP on every host IP, bypassing Traefik - compose can't restrict that. ESPHome's dashboard has no login unless `USERNAME`/`PASSWORD` are set. Keep the host LAN-only.

## Configuration

See `.env.example` for available environment variables.
