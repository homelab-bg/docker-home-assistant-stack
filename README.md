# Home Assistant Stack

Home Assistant with ESPHome, Matter Server and the Everything Presence mmWave Configurator. Optional overlays add Traefik reverse proxying (TLS via Let's Encrypt), a Mosquitto MQTT broker, and a Cloudflare Tunnel.

| File | Adds |
|---|---|
| `docker-compose.yml` | Base: Home Assistant, ESPHome, Matter Server, EP configurator |
| `docker-compose.traefik.yml` | Traefik routing + TLS for Home Assistant, ESPHome, EP configurator |
| `docker-compose.mqtt.yml` | Mosquitto on 1883 (LAN-only, no TLS) - skip if another broker (e.g. an SMHub) already handles MQTT |
| `docker-compose.cloudflared.yml` | Cloudflare Tunnel for external access |

Home Assistant, ESPHome and Matter Server use `network_mode: host` - needed for mDNS/SSDP discovery, Matter and Bluetooth - so they can't join a Docker network. The Traefik overlay routes to them via the host instead (see [Traefik Integration](#traefik-integration)).

## Quick Start

1. Copy environment template:
```bash
cp .env.example .env
```

2. Edit `.env` - set the `*_HOST` values if using Traefik, and `HA_BASE_URL` / `HA_LONG_LIVED_TOKEN` for the EP configurator.

## Deployment Options

Combine the base with whichever overlays you need, e.g.:

```bash
# Standalone
docker compose up -d

# Traefik
docker compose -f docker-compose.yml -f docker-compose.traefik.yml up -d

# Traefik + MQTT + Cloudflare Tunnel
docker compose -f docker-compose.yml -f docker-compose.traefik.yml \
  -f docker-compose.mqtt.yml -f docker-compose.cloudflared.yml up -d
```

Standalone access: Home Assistant at `http://<host>:8123`, ESPHome at `http://<host>:6052`, EP configurator at `http://<host>:42069`.

### Traefik Integration

Requires a Traefik instance on the same host with Let's Encrypt configured (e.g. [docker-traefik-portainer](https://github.com/homelab-bg/docker-traefik-portainer)), and an external `traefik` network.

**Traefik must be able to reach host-network containers.** For those, Traefik's Docker provider forwards to `host.docker.internal`, falling back to `127.0.0.1` (i.e. Traefik's own container). On Linux that only resolves if the Traefik container has:
```yaml
extra_hosts:
  - host.docker.internal:host-gateway
```
docker-traefik-portainer sets this already.

**Home Assistant must trust Traefik**, or it rejects proxied requests with `400: Bad Request`. Add to `ha/config/configuration.yaml`:
```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 172.16.0.0/12   # Docker's default bridge range - confirm Traefik's subnet with `docker network inspect traefik`
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

## TrueNAS Deployment

Put this repo in a dataset (e.g. `/mnt/<pool>/Apps/home-assistant-stack`), with Traefik bound to a secondary IP via docker-traefik-portainer's TrueNAS overlay. Then **Apps → Discover Apps → Install via YAML**, listing the base plus whichever overlays you want:
```yaml
include:
  - path:
      - /mnt/<pool>/Apps/home-assistant-stack/docker-compose.yml
      - /mnt/<pool>/Apps/home-assistant-stack/docker-compose.traefik.yml
      #- /mnt/<pool>/Apps/home-assistant-stack/docker-compose.mqtt.yml
      #- /mnt/<pool>/Apps/home-assistant-stack/docker-compose.cloudflared.yml

# Web Portal buttons in the TrueNAS Apps UI. These go here, not in the compose
# files, because `include:` drops top-level x- extensions from included files.
# Use literal hostnames: .env isn't applied to this file.
x-portals:
  - {name: Home Assistant, scheme: https, host: ha.yourdomain.com, port: 443, path: /}
  - {name: ESPHome, scheme: https, host: esphome.yourdomain.com, port: 443, path: /}
```

The dataset is the project directory, so `.env`, `./secrets` and the app data directories all resolve inside it. Set `MATTER_INTERFACE` in `.env` to the TrueNAS NIC (`ip -br link`) - the `eth0` default suits a VM.

## Security

⚠️ Host networking means Home Assistant (8123) and ESPHome (6052) stay reachable over plain HTTP on every host IP, bypassing Traefik - compose can't restrict that. ESPHome's dashboard has no login unless `USERNAME`/`PASSWORD` are set. Keep the host LAN-only.

## Configuration

See `.env.example` for available environment variables.
