# NetFoundry Autonomous Edge Router

A containerized [NetFoundry](https://netfoundry.io) / OpenZiti Edge Router that self-registers, manages its own DNS, and optionally self-updates — all from a single `compose.yml`.

---

## Requirements

- Docker + Docker Compose v2
- Ubuntu 22.04 or 24.04 (host)
- A NetFoundry registration key

**One-time host prep:**
```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
sudo chmod 755 /etc/systemd/resolved.conf.d
```

---

## Quick Start

**1. Clone the repo**
```bash
git clone https://github.com/nicfragale/autonomous-router.git
cd autonomous-router
```

**2. Create a `.env` file**
```bash
cp .env.example .env
```

Edit `.env` and set your registration key:
```env
REG_KEY=your-netfoundry-registration-key
```

**3. Start the router**
```bash
docker compose up -d
```

On first run the router will register with NetFoundry using your `REG_KEY`, download its identity and certificates, and start automatically. Subsequent restarts skip registration.

**4. Check the logs**
```bash
docker compose logs -f
```

---

## Configuration

All settings are controlled via environment variables in your `.env` file. The compose file provides defaults for everything except `REG_KEY`.

| Variable | Default | Description |
|---|---|---|
| `ZITI_CONTAINER_NAME` | `autonomous-er` | Name for the container |
| `REG_KEY` | *(required on first run)* | NetFoundry registration key |
| `TUNNEL_MODE` | `auto` | Ziti tunnel mode (`auto` recommended) |
| `DISABLE_AUTO_UPDATE` | `true` | Prevent mid-run Ziti binary upgrades |
| `ZITI_VERSION_OVERRIDE` | *(unset)* | Pin a specific Ziti version e.g. `v1.7.2` |
| `ZITI_DNS_RANGE` | `100.65.0.0/24` | IP range for overlay service addresses |
| `ZITI_DNS_UPSTREAM` | `udp://1.1.1.1:53` | Upstream resolver inside Ziti config |
| `ZITI_DNS_FALLBACK` | `1.1.1.1` | Fallback resolver if Ziti DNS can't answer |
| `HOSTS_ENTRIES` | `INTERFACE_ASSIGNED` | Static `/etc/hosts` injection (see below) |

### HOSTS_ENTRIES

Controls what gets injected into `/etc/hosts` before Ziti starts. These entries are resolved locally by the OS and are **never** forwarded to the overlay.

| Format | Example | Behavior |
|---|---|---|
| IP + hostname | `192.168.1.10 db.internal` | Written as-is |
| Hostname only | `myhost.corp` | Forward DNS lookup, resolved IP written |
| IP only | `10.20.30.84` | Reverse PTR lookup, all records written |
| `INTERFACE_ASSIGNED` | *(default)* | Auto-detects host LAN IP + reverse lookup |

---

## DNS Behavior

With default settings, **all DNS on the host** is routed through the Ziti DNS server:

```
Host process
    └── nsswitch (checks /etc/hosts first)
          └── systemd-resolved → Ziti DNS server
                ├── Overlay record?  → returns overlay IP  (traffic tunnels through Ziti)
                ├── Unknown record?  → forwards to 1.1.1.1 (traffic exits normally)
                └── Unreachable?     → falls back to ZITI_DNS_FALLBACK
```

`.local` queries bypass the mDNS stack and flow through Ziti DNS as well (`ZITI_DNS_DISABLE_MDNS=true`).

---

## Stopping & Cleanup

```bash
# Stop the router (graceful 30s shutdown)
docker compose down
```

On shutdown the entrypoint automatically:
- Stops the Ziti process
- Removes the systemd-resolved drop-in config
- Restarts systemd-resolved to restore original DNS

Router identity and certs are persisted in `./ziti_router` and survive restarts.

---

## Image

Published to GitHub Container Registry:
```
ghcr.io/nicfragale/autonomous-router:latest
```



---

### Startup & DNS Flow

![Autonomous Edge Router Flow](flow.jpg)
