# PIRates-router
# FriendlyWrt + Traefik + Authelia + Headscale

Self-contained edge routing, zero-trust reverse proxy, and self-hosted mesh VPN stack running on a **FriendlyELEC NanoPi R6S** (RK3588S).

This setup eliminates all external SaaS/cloud control-plane dependencies:
* **Routing & Firewall:** Bare-metal **FriendlyWrt** handling multi-gigabit WAN/LAN routing, DHCP static leases, and edge firewalling.
* **Public Reverse Proxy:** **Traefik** (Docker) terminating TLS on ports `80`/`443` and routing traffic to internal lab hosts (`192.168.2.x`).
* **Identity & SSO (IdP):** **Authelia** (Docker) providing 2FA `forwardAuth` for public web services and acting as the **OIDC provider** for VPN authentication.
* **Self-Hosted VPN Control Plane:** **Headscale** (Docker) with an embedded DERP relay replacing Tailscale's public servers, paired with a native **`tailscaled`** daemon on the R6S host for LAN subnet routing.

---

## Architecture Overview

```text
                        [ Public Internet ]
                                 │
                   ┌─────────────┴─────────────┐
                   │   NanoPi R6S (WAN Edge)   │
                   │  FriendlyWrt + Firewall   │
                   └─────────────┬─────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │ (TCP 80/443)          │ (UDP 3478)            │ (WireGuard / Mesh)
         ▼                       ▼                       ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│     Traefik     │     │    Headscale    │     │   tailscaled    │
│ (Reverse Proxy) │◄───►│ (Control + DERP)│◄───►│ (Host Daemon)   │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         ├───────────────────────┤ (OIDC Auth)           │ (Subnet Route:
         ▼                       ▼                       │  192.168.2.0/24)
┌─────────────────┐                                      │
│    Authelia     │                                      │
│ (2FA + OIDC IdP)│                                      │
└────────┬────────┘                                      │
         │ (Authorized Traffic)                          │
         ▼                                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Home Lab LAN (192.168.2.0/24)                │
│         Proxmox / Docker Hosts / NAS / Internal Services        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Repository Structure

Deploy this repository into `/opt/docker/edge-stack` on the R6S (located on the large `userdata` partition rather than the smaller root overlay filesystem).

```text
.
├── docker-compose.yml              # Traefik, Authelia, and Headscale services
├── .env.example                    # Environment variables template (domains, secrets)
├── host/
│   └── setup-friendlywrt.sh        # Host prep: uhttpd port move, kmod-tun, tailscale
├── traefik/
│   ├── traefik.yml                 # Static config (entrypoints, Let's Encrypt, providers)
│   ├── acme.json                   # TLS certificate storage (chmod 600, gitignored)
│   └── dynamic/
│       ├── middlewares.yml         # Authelia forwardAuth & security headers
│       └── lab-services.yml        # Routes to backend lab PCs (192.168.2.x)
├── authelia/
│   ├── configuration.yml           # 2FA policies, forwardAuth, and Headscale OIDC client
│   └── users_database.yml          # Local user accounts & password hashes (gitignored)
└── headscale/
    └── config.yaml                 # Headscale control plane, OIDC issuer & embedded DERP
```

---

## Prerequisites

1. **Hardware & OS:**
   * NanoPi R6S flashed with FriendlyELEC's official **FriendlyWrt (Docker)** `eFlasher` image.
2. **DNS Records:**
   * `://yourdomain.com` -> Public WAN IP (or Dynamic DNS CNAME)
   * `://yourdomain.com` -> Public WAN IP
   * `*.yourdomain.com` -> Public WAN IP (for exposed lab services)
3. **LAN IP Management:**
   * Assign **Static DHCP Leases** in FriendlyWrt (**Network > DHCP and DNS > Static Leases**) to all backend lab servers (`192.168.2.x`).

---

## Phase 1: Host Preparation (FriendlyWrt)

Run these steps via SSH on the NanoPi R6S before starting Docker:

### 1. Move LuCI Web UI off Ports 80/443
Free standard HTTP/HTTPS ports for Traefik:
```bash
uci set uhttpd.main.listen_http='0.0.0.0:8080'
uci set uhttpd.main.listen_https='0.0.0.0:8443'
uci commit uhttpd
/etc/init.d/uhttpd restart
```
*(LuCI is now accessible at `http://192.168.2.1:8080`.)*

### 2. Configure Firewall & Disable Flow Offloading
In LuCI (**Network > Firewall**):
* **General Settings:** Ensure **Software/Hardware Flow Offloading** is **disabled** (prevents packet drops between Docker bridges and WAN/LAN).
* **Zones:** Set the `docker` zone **Forward** policy to **Accept**.
* **Traffic Rules:** Open incoming `WAN` traffic to **This device**:
  * `TCP 80, 443` (Traefik HTTP/HTTPS)
  * `UDP 3478` (Headscale embedded DERP relay)

### 3. Install Kernel TUN Module & Native Tailscale Client
```bash
opkg update
opkg install kmod-tun tailscale
/etc/init.d/tailscale enable
/etc/init.d/tailscale start
```

---

## Phase 2: Deploy the Docker Stack

1. Clone this repository into `/opt/docker/edge-stack`:
   ```bash
   mkdir -p /opt/docker && cd /opt/docker
   git clone <your-repo-url> edge-stack && cd edge-stack
   ```
2. Prepare secrets and file permissions:
   ```bash
   cp .env.example .env
   touch traefik/acme.json && chmod 600 traefik/acme.json
   ```
3. Generate OIDC client secrets and password hashes using Authelia:
   ```bash
   docker run --rm authelia/authelia:latest authelia crypto hash generate argon2 --password 'YourStrongPassword'
   docker run --rm authelia/authelia:latest authelia crypto rand --length 64 --charset alphanumeric
   ```
4. Populate `authelia/configuration.yml`, `authelia/users_database.yml`, and `headscale/config.yaml`, then launch the stack:
   ```bash
   docker compose up -d
   ```

---

## Phase 3: Connect Host & Client Devices to Headscale

1. **Create a Headscale user namespace:**
   ```bash
   docker exec headscale headscale users create lab-admin
   ```
2. **Register the NanoPi R6S host and advertise your LAN subnet:**
   ```bash
   tailscale up --login-server=https://://yourdomain.com --advertise-routes=192.168.2.0/24
   ```
   Complete the login prompt in your browser via Authelia (`https://://yourdomain.com`).
3. **Approve the R6S subnet route in Headscale:**
   ```bash
   docker exec headscale headscale nodes list
   docker exec headscale headscale routes list
   docker exec headscale headscale routes enable -r <ROUTE_ID>
   ```
4. **Connect remote clients (Laptop / Mobile):**
   * Point your Tailscale client's custom control server URL to `https://://yourdomain.com` and authenticate via Authelia.

---

## Adding New Lab Services

To expose a service running on another lab PC (e.g., `192.168.2.50:8096`) behind Traefik + Authelia 2FA, add a router and service entry to `traefik/dynamic/lab-services.yml`. Traefik hot-reloads changes automatically with zero downtime.

---

## Security Notes

* Never commit `.env`, `traefik/acme.json`, `authelia/users_database.yml`, or Headscale private keys (`noise_private.key` / `derp_server_private.key`) to Git.
* Keep FriendlyWrt LuCI (`:8080`) and SSH (`:22`) blocked on the `WAN` zone—access them exclusively over LAN or your Headscale mesh network.

