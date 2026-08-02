# 📂 Synology NAS Documentation & Self-Hosting Guides

Welcome to the **Synology Docs** repository! This repository contains comprehensive, battle-tested, step-by-step guides for self-hosting applications and configuring advanced networking solutions on **Synology DSM 7.2+** using **Container Manager (Docker)**.

---

## 📚 Documentation Index

| Guide | Description | Highlights |
| :--- | :--- | :--- |
| ✈️ **[Plane on Synology DSM 7.2](Plane_Synology_Guide.md)** | Step-by-step guide to deploying Plane (open-source project management tool) locally via Container Manager. | Custom `.env` template, sanitized `docker-compose.yaml`, Synology CLI bug workaround, database & minio storage setup. |
| ☁️ **[Plane + Cloudflare Tunnel Setup (5G/CGNAT)](Synology_Plane_Cloudflare_Setup.md)** | End-to-end setup for exposing Plane securely to the internet behind 5G or CGNAT without port forwarding. | Cloudflare Zero Trust Tunnels, custom domain routing, HTTPS, and local network DNS filtering workaround. |
| 🌐 **[ZeroTier Behind Strict Firewalls](zerotier_synology_guide.md)** | Deploying ZeroTier SD-WAN on Synology Docker under restrictive networks or enterprise firewalls. | Persistent TUN driver (`/dev/net/tun`) setup, folder permission fixes, TCP relay fallback, and routing configs. |

---

## 🛠 Prerequisites & Environment

Most guides in this repository assume the following setup on your Synology NAS:

- **OS**: Synology DiskStation Manager (DSM) **7.2 or higher**
- **Packages**:
  - **Container Manager** (native Docker management)
  - **Text Editor** (via Package Center)
- **Access**:
  - **SSH Access** enabled (*Control Panel > Terminal & SNMP > Enable SSH service*)
  - **File Station** access for managing `/volume1/docker/` shared folders

---

## 🚀 Key Self-Hosting Scenarios

### 1. Project Management with Plane
Deploy Plane locally on port `8090` using a clean Docker Compose file tailored specifically for Synology DSM Container Manager.
- **Local Access**: See [Plane_Synology_Guide.md](Plane_Synology_Guide.md)
- **Public Domain Access (CGNAT/5G Bypass)**: See [Synology_Plane_Cloudflare_Setup.md](Synology_Plane_Cloudflare_Setup.md)

### 2. Remote Access via ZeroTier VPN
Bypass CGNAT, strict NATs, or restricted UDP ports by running ZeroTier in a privileged container with persistent TUN module injection.
- **ZeroTier Guide**: See [zerotier_synology_guide.md](zerotier_synology_guide.md)

---

## 💡 Quick Troubleshooting Tips

- **Cloudflare Tunnel `i/o timeout` (Port 7844)**:
  - *Cause*: Local network DNS filtering blocking `*.argotunnel.com`.
  - *Fix*: Manually set Synology DNS servers to Cloudflare `1.1.1.1` and `1.0.0.1` (*Control Panel > Network > General*) and restart `cloudflared-tunnel`.
- **ZeroTier Missing Virtual Network Driver (`/dev/net/tun`)**:
  - *Cause*: Synology does not load `tun.ko` automatically on startup.
  - *Fix*: Use the boot startup script provided in `zerotier_synology_guide.md` under `/usr/local/etc/rc.d/tun.sh`.
- **Plane Container Manager Build Errors**:
  - *Cause*: GUI variable interpolation or syntax incompatibilities in raw Compose files.
  - *Fix*: Use the hardcoded, DSM-compatible Compose template provided in the guides.

---

## 📜 License

This repository is licensed under the [MIT License](LICENSE).
