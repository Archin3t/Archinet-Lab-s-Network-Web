# Segmented Homelab Network

> **A documented, production-style VLAN-segmented home network built around a Sophos XG 115w, HP 1920-24G, and OPNsense.**
> Designed to be reproducible, hardware-agnostic, and easy to expand.

This repository documents the **architecture**, **design decisions**, and **deployment strategy** behind my personal network build. While the reference platform uses a Sophos 115w and HP 1920-24G, the same blueprint applies to any pfSense/OPNsense-capable firewall and any managed switch supporting 802.1Q VLANs.

Unlike an installation guide, this repository serves as the **platform documentation** for the network. Individual services (homelab, media stack, NAS) are documented in their own dedicated repositories.

---

# Companion Repositories

[#companion-repositories](#companion-repositories)

To keep documentation organized, applications and services are split into dedicated repositories.

| Repository | Covers |
|---|---|
| **[ZimaBoard-2-Homelab](https://github.com/Archin3t/ZimaBoard-2-Homelab)** | Proxmox homelab platform, hardware, and design philosophy |
| **[Homelab-Media-Stack](https://github.com/Archin3t/Homelab-Media-Stack)** | Jellyfin, Sonarr, Radarr, Prowlarr, qBittorrent, Kavita |
| **[ZimaBoard-NAS](https://github.com/Archin3t/ZimaBoard-NAS)** | NAS, storage, SMB/NFS shares, backups |

---

# Reference Hardware

[#reference-hardware](#reference-hardware)

| Component | Specification |
|---|---|
| **Firewall/Router** | Sophos XG 115w |
| **Routing OS** | OPNsense |
| **Switch** | HP (Aruba) 1920-24G (JG924A) |
| **Access Point** | Cudy TR3000 (OpenWrt, bridge mode) |
| **DMZ Host** | Lenovo ThinkCentre M900 Tiny |

Although this documentation references specific hardware, the overall architecture applies equally well to:

- Any OPNsense/pfSense-capable firewall
- Any 802.1Q-capable managed switch
- Any OpenWrt-flashable AP for bridge-mode wireless

Scale VLAN count and subnet sizing as needed while keeping zone isolation intact.

---

# What It Does

[#what-it-does](#what-it-does)

```
                        Internet
                            │
                            ▼
                    Sophos 115w / OPNsense
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
       MGMT (VLAN 10)  LAB (VLAN 20)  DMZ (physical)
       10.10.0.0/24    10.10.10.0/24  10.10.99.0/24
              │             │             │
              ▼             ▼             ▼
        Switch/AP/FW    Wired + WiFi    M900
        admin access    lab devices     (web-facing)

──────────────────────────────────────────────

 Isolation enforced at the firewall:

 LAB  → MGMT   BLOCKED
 LAB  → DMZ    BLOCKED
 DMZ  → MGMT   BLOCKED
 DMZ  → LAB    BLOCKED
 MGMT → any    ALLOWED
```

---

# Topology

[#topology](#topology)

```
WAN (Internet)
  │
Sophos 115w / OPNsense
  ├── MGMT (VLAN 10) ── 10.10.0.0/24   ── 10.10.0.1
  ├── LAB  (VLAN 20) ── 10.10.10.0/24  ── 10.10.10.1
  └── DMZ  (physical)── 10.10.99.0/24  ── 10.10.99.1
       │
HP 1920-24G
  ├── Port 1     → Trunk (VLAN 10 + 20 tagged) → OPNsense
  ├── Port 2–23  → VLAN 20 untagged (LAB)
  ├── Port 4     → Cudy TR3000 (AP, bridged to VLAN 20)
  └── Port 24    → VLAN 10 untagged (MGMT — admin access only)
```

---

# VLAN / Subnet Reference

[#vlan-subnet-reference](#vlan-subnet-reference)

| VLAN | Name | Subnet | Gateway | DHCP Range | Purpose |
|---|---|---|---|---|---|
| 10 | MGMT | `10.10.0.0/24` | `10.10.0.1` | `.200–.249` | Firewall/switch/AP administration — port 24 only |
| 20 | LAB | `10.10.10.0/24` | `10.10.10.1` | `.150–.249` | Wired lab devices + WiFi |
| — | DMZ | `10.10.99.0/24` | `10.10.99.1` | `.200–.249` | Internet-facing services, physical Sophos port |

---

# Firewall Isolation Policy

[#firewall-isolation-policy](#firewall-isolation-policy)

| Source | Destination | Action |
|---|---|---|
| MGMT | any | Allow |
| LAB | MGMT | **Block** |
| LAB | DMZ | **Block** |
| LAB | Internet | Allow |
| DMZ | MGMT | **Block** |
| DMZ | LAB | **Block** |
| DMZ | Internet | Allow (restricted ports only) |
| WAN | DMZ | Allow (80/443 only) |

Rule order matters — block rules sit **above** broad allow rules, since OPNsense evaluates top-down and stops at first match.

---

# Access Point

[#access-point](#access-point)

- **Device:** Cudy TR3000, flashed to OpenWrt
- **Mode:** Bridge/AP — routing/NAT/DHCP disabled
- **LAN IP:** Static, in-subnet with LAB (`10.10.10.2`)
- **Uplink:** HP port 4 (VLAN 20 untagged)
- **DNS/DHCP:** Provided entirely by OPNsense (Unbound + Kea) — no secondary DHCP server on the AP

---

# Known Gotchas (HP 1920-24G)

[#known-gotchas](#known-gotchas)

- No physical reset button — factory reset via bridging ports 1 + 2 during power-on
- Console port: 38400 baud, 8N1 (not the more common 9600)
- Web UI can hang indefinitely if the management session rides through the port/VLAN currently being modified — always manage from a port that isn't part of the change being made
- Newly created VLAN interfaces on OPNsense are **not** automatically added to DHCP or Unbound's listening interface list — this must be done manually for each new VLAN or you'll get link-local (APIPA) addresses or DNS failure with a live IP

---

# Repository Documentation

[#repository-documentation](#repository-documentation)

| File | Purpose |
|---|---|
| **DOCUMENTATION.md** | Complete network architecture and technical reference |
| **CREATION-GUIDE.md** | Recommended build order from bare hardware to finished segmented network |
| **USER-GUIDE.md** | Day-to-day administration, adding devices, troubleshooting |
| **SECURITY.md** | Isolation policy rationale, credential redaction, hardening notes |

---

# Design Philosophy

[#design-philosophy](#design-philosophy)

This network follows a few simple principles:

- Default deny between zones, explicit allow only
- Management plane reachable from exactly one physical port
- No service should be able to discover a network it has no business knowing exists
- Document everything, including the mistakes
- Prefer stability over cleverness
- Build for learning without sacrificing real isolation

The goal is a network where a compromised host in one zone has no path to anything outside its own zone.

---

# Security

[#security](#security)

This repository intentionally excludes sensitive information.

Examples include:

- Passwords
- API Keys / Tokens
- WireGuard configs and keys
- Public IP addresses
- Physical location details
- Real device serial numbers / MACs

Documentation uses placeholders instead.

```
<FIREWALL_ADMIN_IP>
<SWITCH_MGMT_IP>
<USERNAME>
<PASSWORD>
```

---

# About

[#about](#about)

This repository documents my personal network segmentation build as both a reference implementation and an educational resource — how to take a flat home network and turn it into properly isolated management, lab, and DMZ zones using consumer/prosumer hardware.

---

## Credits

[#credits](#credits)

Created and maintained by **Archin3t**

- 🌐 GitHub: https://github.com/Archin3t
- ▶️ YouTube: https://www.youtube.com/@Archinet-Labs
- 📸 Instagram: https://www.instagram.com/Archin3t

---

© 2026 Archin3t. All Rights Reserved.

If this repository helped you segment your own network, create documentation, make a video, or publish a guide, **please provide visible credit and link back to this repository.**
