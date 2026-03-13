# JACOSShield

> **A hardened, security-focused firewall OS built on pfSense CE 2.7.2 — FreeBSD 14.0-RELEASE, amd64**

[![Build](https://img.shields.io/badge/build-JACOSShield--CE--2.7.2--RELEASE-blue)](https://github.com/iposup-a3/JACOSShield)
[![Base](https://img.shields.io/badge/base-pfSense%20CE%202.7.2-orange)](https://github.com/pfsense/pfsense)
[![FreeBSD](https://img.shields.io/badge/FreeBSD-14.0--RELEASE-red)](https://www.freebsd.org/)
[![Arch](https://img.shields.io/badge/arch-amd64-lightgrey)](https://github.com/iposup-a3/JACOSShield)
[![License](https://img.shields.io/badge/license-Apache%202.0-green)](LICENSE)

---

## Table of Contents

- [Overview](#overview)
- [What is JACOSShield](#what-is-jacosshield)
- [Security Philosophy](#security-philosophy)
- [ISO Details](#iso-details)
- [Build Environment](#build-environment)
- [Repository Structure](#repository-structure)
- [How to Build](#how-to-build)
- [Installation](#installation)
- [VirtualBox / VM Testing](#virtualbox--vm-testing)
- [Package Strategy](#package-strategy)
- [Default Configuration](#default-configuration)

---

## Overview

JACOSShield is a custom firewall operating system derived from pfSense Community Edition 2.7.2. It is built on top of Netgate's open-source pfSense build toolchain targeting hardened enterprise firewall deployments.

This repository contains all build system modifications, configuration templates, package repository definitions, and product assets that form the JACOSShield CE 2.7.2 release.

---

## What is JACOSShield

JACOSShield is a firewall appliance OS designed with the following goals:

- **Security-first:** Reduced attack surface by excluding high-risk packages that open remote access vectors
- **Reliability:** Built on the stable FreeBSD 14.0-RELEASE base with pfSense CE 2.7.2 codebase
- **Maintainability:** All build customizations are documented and version-controlled in this repository
- **Self-hosted packages:** Packages are built and served locally via Poudriere and nginx — no dependency on Netgate package servers

---

## Security Philosophy

JACOSShield is built around the principle that **a firewall should only contain what is strictly necessary to function as a firewall.**

**Remote access packages** — OpenVPN, StrongSwan, and similar VPN daemons can be exploited to establish tunnels directly into the firewall. These are excluded entirely.

**Network discovery tools** — Nmap, Netcat, and similar tools have no legitimate operational role on a production firewall.

**Protocol-opening daemons** — UPnP (miniupnpd) automatically opens firewall ports on request from LAN devices — fundamentally incompatible with a controlled security posture.

**Unnecessary file sharing** — Samba and similar Windows file sharing stacks have no role on a firewall appliance.

**Extra web servers** — Only the nginx instance required by the WebGUI is included. All other HTTP daemons are excluded.

---

## ISO Details

| Field | Value |
|---|---|
| Product | JACOSShield CE |
| Version | 2.7.2-RELEASE |
| Architecture | amd64 |
| Base | pfSense CE 2.7.2 / FreeBSD 14.0-RELEASE |
| Build Date | March 12, 2026 |
| ISO filename | `JACOSShield-CE-2.7.2-RELEASE-amd64.iso` |
| ISO size (compressed) | 2.4 GB |
| ISO size (uncompressed) | 4.3 GB |
| Packages built | 437 (via Poudriere) |
| Packages embedded in ISO | 4 core packages |

### Packages Embedded in ISO

| Package | Version | Purpose |
|---|---|---|
| `JACOSShield-ce` | 2.7.2 | Main CE meta-package |
| `JACOSShield-rc` | 2.7.2 | RC startup scripts |
| `drm-510-kmod` | 5.10.163_8 | DRM kernel modules |
| `pkg` | 1.20.8_3 | Package manager |

---

## Build Environment

| Parameter | Value |
|---|---|
| Build Host OS | FreeBSD 14.0-RELEASE |
| Architecture | amd64 |
| Working directory | `/root/pfsense` |
| Branch | `RELENG_2_7_2` |
| Kernel config | `JACOSShield` |
| Package signing | `/root/sign/sign.sh` + `/root/sign/repo.key` |
| Package server | `http://127.0.0.1/packages` (local nginx) |
| Poudriere jail | `JACOSShield_v2_7_2_amd64` |
| Poudriere ports | `JACOSShield_v2_7_2` |
| Build logs | `/root/logs/v1.7/` |

---

## Repository Structure

```
JACOSShield/
├── build.conf.bak.iso
├── src/
│   ├── conf.default/
│   │   └── config.xml                         # Default firewall configuration
│   └── usr/local/share/JACOSShield/
│       └── keys/pkg/
│           ├── revoked/                        # Revoked package signing keys
│           └── trusted/                        # Trusted package signing keys
└── tools/
    ├── builder_common.sh                       # Core build logic
    ├── builder_defaults.sh                     # Build defaults and signing config
    ├── conf/
    │   └── pfPorts/
    │       └── poudriere_bulk                  # Package list for Poudriere
    └── templates/
        ├── core_pkg/
        │   └── rc/metadir/
        │       ├── +DESC
        │       └── +MANIFEST
        └── pkg_repos/
            ├── JACOSShield-repo.conf           # Production repo config
            ├── JACOSShield-repo-devel.conf     # Devel repo config
            └── JACOSShield-repo-staging.conf   # Staging repo config
```

---

## How to Build

> **Prerequisites:** FreeBSD 14.0-RELEASE build host, 176 GB+ disk (ZFS recommended), 2+ CPUs, Poudriere installed, nginx running on port 80.

### Step 1 — Clone this repository
```sh
git clone https://github.com/iposup-a3/JACOSShield.git /root/pfsense
cd /root/pfsense
```

### Step 2 — Build world and kernel (one-time, ~3–4 hours)
```sh
cd /root/pfsense
./build.sh --skip-final-rsync buildworld buildkernel
```

### Step 3 — Build packages with Poudriere (~6–8 hours)
```sh
poudriere bulk -j JACOSShield_v2_7_2_amd64 -p JACOSShield_v2_7_2 \
  -f /root/pfsense/tools/conf/pfPorts/poudriere_bulk
```

Monitor:
```sh
poudriere status -j JACOSShield_v2_7_2_amd64 -p JACOSShield_v2_7_2
```

### Step 4 — Build the ISO
```sh
cd /root/pfsense
unset SRCCONF FREEBSD_SRC_DIR MAKEOBJDIRPREFIX

daemon -f -o /root/logs/v1.7/iso_build.log \
  bash -c 'cd /root/pfsense && export NO_BUILDWORLD=YES NO_BUILDKERNEL=YES \
  DO_NOT_SIGN_PKG_REPO=YES && ./build.sh --no-cleanobjdir --skip-final-rsync iso'

tail -f /root/logs/v1.7/iso_build.log | grep ">>>"
```

### Step 5 — Locate and decompress the ISO
```sh
find /root/pfsense/tmp -name "*.iso.gz"
gunzip /root/pfsense/tmp/JACOSShield/installer/JACOSShield-CE-2.7.2-RELEASE-amd64.iso.gz
```

### Resume Build (if world/kernel already built)
```sh
sh /tmp/finish_iso.sh
```

---

## Installation

### System Requirements

| Component | Minimum | Recommended |
|---|---|---|
| CPU | 64-bit (amd64) | 2+ cores |
| RAM | 512 MB | 2 GB |
| Storage | 8 GB | 20 GB |
| Network interfaces | 2 NICs | 2+ NICs |
| Boot mode | BIOS | BIOS |

### Installation Steps

1. Attach the ISO to the target machine as optical drive or USB
2. Boot from the ISO
3. Select **Install JACOSShield** from the boot menu
4. Choose **Auto (ZFS)** partitioning
5. Select **stripe** as the pool type for single-disk installs
6. Set the ZFS pool name to `JACOSShield`
7. Confirm **GPT (BIOS)** partition scheme
8. Allow installation to complete and reboot
9. Remove the ISO on reboot
10. Access the WebGUI at `https://192.168.1.1`

> Default credentials: `admin` / `pfsense` — **change immediately after first login.**

---

## VirtualBox / VM Testing

| Setting | Value |
|---|---|
| Type | BSD |
| Version | FreeBSD (64-bit) |
| RAM | 2048 MB minimum |
| CPU | 2 cores |
| Storage controller | **SATA** (not IDE) |
| Disk | 20 GB, VDI, dynamically allocated |
| Network Adapter 1 | Bridged Adapter (WAN) |
| Network Adapter 2 | Internal Network (LAN) |
| Enable EFI | **Unchecked** — JACOSShield uses BIOS boot |

---

## Package Strategy

### Core packages included

| Package | Role |
|---|---|
| `pkg` | Package manager |
| `php82` + extensions | WebGUI backend |
| `nginx` | WebGUI web server |
| `unbound` | DNS resolver |
| `isc-dhcp44-server` | DHCP server |
| `filterlog` | Firewall log parser |
| `pftop` | PF real-time monitor |
| `ca_root_nss` | TLS certificate authorities |
| `curl` | HTTP client |
| `sqlite3` | Database |
| `JACOSShield-ce` | Product meta-package |

### Excluded packages

| Package | Reason |
|---|---|
| `openvpn` | Remote access vector |
| `strongswan` | Remote access vector |
| `miniupnpd` | Automatic port-opening — security risk |
| `nmap` | No operational role on production firewall |
| `netcat` / `ncat` | Pivot tool risk |
| `samba416` | No role on a firewall appliance |
| `lighttpd` / `apache24` | Unnecessary attack surface |

---

## Default Configuration

After installation, JACOSShield boots with the following default network configuration:

| Interface | Assignment | IP Address |
|---|---|---|
| First NIC | WAN | DHCP |
| Second NIC | LAN | 192.168.1.1/24 |

WebGUI is accessible at `https://192.168.1.1` from the LAN interface.

---

*JACOSShield CE 2.7.2 — Built March 2026*
*JACOSShield is an independent derivative of pfSense CE. Not affiliated with or endorsed by Netgate.*
```
