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

### General

| Variable | Default | Description |
|---|---|---|
| `ZITI_CONTAINER_NAME` | `autonomous-router` | Docker container name. Set this in your `.env` to give the container a custom name. |
| `REG_KEY` | *(required on first run)* | NetFoundry registration key. Ignored once `certs/cert.pem` exists. |
| `TUNNEL_MODE` | `auto` | Ziti tunnel mode. `auto` enables the tunnel listener (required for DNS routing). Set to `host` or unset for host-only mode. **Note:** this is written into `config.yml` at registration time. Changing it after the router is already registered has no effect — delete `./ziti_router` and re-register with a new key to apply the change. |
| `VERBOSE` | *(unset)* | Set to any non-empty value to pass `-v` to `ziti router run`. |
| `HTTPS_PROXY` | *(unset)* | Proxy URL for outbound connections e.g. `http://proxy.corp:3128`. |
| `ADVERTISE_ADDRESS` | *(unset)* | Override the advertised edge listener address. Required when the host's public IP differs from its LAN IP. Format: `IP:PORT` e.g. `203.0.113.10:443`. |

### Update Control

| Variable | Default | Description |
|---|---|---|
| `DISABLE_AUTO_UPDATE` | `false` | Set to `true` to skip controller-version checks entirely. Ziti starts once and runs until the container stops. The crash-restart supervisor still runs. |
| `ZITI_VERSION_OVERRIDE` | *(unset)* | Pin the router binary to a specific version, bypassing all version-match checks. Format: `vMAJOR.MINOR.PATCH` e.g. `v1.7.2`. Setting both this and `DISABLE_AUTO_UPDATE=true` gives a fully static, no-loop deployment. |

### DNS Setup (`ZITI_DNS_CONFIGURE=true`)

| Variable | Default | Description |
|---|---|---|
| `ZITI_DNS_CONFIGURE` | `true` | Set to `true` to enable DNS setup at startup. Writes a systemd-resolved drop-in and restarts resolved. Requires the `systemd` volume mounts below. |
| `ZITI_DNS_MODE` | `all` | `all` — route **every** DNS query through Ziti DNS (recommended). `domains` — route only the domains listed in `ZITI_DNS_DOMAINS`. |
| `ZITI_DNS_DOMAINS` | *(unset)* | Space-separated list of domains to route through Ziti DNS. Only used when `ZITI_DNS_MODE=domains`. e.g. `lan ziti corp internal`. |
| `ZITI_DNS_IP` | *(auto-detected)* | IP address of the Ziti DNS server. Auto-detected from the host's default LAN route when unset. |
| `ZITI_DNS_FALLBACK` | *(unset)* | Fallback resolver written into the systemd-resolved drop-in for queries Ziti DNS cannot answer. Auto-detected from the host's current resolver when unset. |
| `ZITI_DNS_DISABLE_MDNS` | `false` | Set to `true` to add `MulticastDNS=no` to the resolved drop-in. Required when your DNS server serves `.local` records — without this, systemd-resolved intercepts `.local` on the mDNS stack (RFC 6762) before the query ever reaches Ziti DNS, returning SERVFAIL. |
| `ZITI_DNS_WAIT_TIMEOUT` | `60` | Seconds to wait for Ziti DNS to become reachable after Ziti starts. Increase if your controller is slow to provision the DNS listener. |
| `ZITI_DNS_HEALTH_THRSH` | `3` | Consecutive failed DNS probes before the resolver reverts to host DNS and Ziti is restarted. Each probe runs once per supervisor loop iteration (~60 s), so the default triggers after ~3 min of unresponsive Ziti DNS. |

### Ziti Config Patches (`ZITI_DNS_CONFIGURE=true`)

| Variable | Default | Description |
|---|---|---|
| `ZITI_DNS_RANGE` | *(unset)* | CIDR written into `config.yml` as `dnsSvcIpRange` (overlay service address pool). If unset will default to `100.64.0.0/10`. |
| `ZITI_DNS_UPSTREAM` | *(auto-detected)* | Upstream DNS written into `config.yml` as `dnsUpstream` (used by the Ziti DNS server for non-overlay lookups). Auto-detected from the host's current resolver when unset (same source as `ZITI_DNS_FALLBACK`, formatted as `udp://IP:53`). **When setting explicitly, the format must include protocol and port:** `udp://IP:PORT` or `tcp://IP:PORT` — e.g. `udp://1.1.1.1:53`. |

### Hosts Injection

| Variable | Default | Description |
|---|---|---|
| `HOSTS_ENTRIES` | `INTERFACE_ASSIGNED` | Static records injected into `/etc/hosts` before Ziti starts. Resolved locally by the OS — never forwarded to the overlay. Separated by semicolons or newlines. Block is removed automatically on container shutdown. To modify the **host's** `/etc/hosts` (not just the container's), add `- /etc/hosts:/etc/hosts` to your compose volumes. |

**Supported `HOSTS_ENTRIES` formats:**

| Format | Example | Behavior |
|---|---|---|
| IP + hostname | `192.168.1.10 db.internal alias` | Written as-is |
| Hostname only | `myhost.corp` | Forward DNS lookup before Ziti starts; resolved IP is written |
| IP only | `10.20.30.84` | Reverse PTR lookup; all PTR records written as aliases |
| `INTERFACE_ASSIGNED` | *(default)* | Auto-detects host LAN IP at runtime, then reverse-lookups it |

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
