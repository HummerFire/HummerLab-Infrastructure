# 📡 Networking Layer — HummerLab

Documentación completa de red: firewalls HA, DNS redundante, BMC/IPMI y red privada inter-server.

---

## 🗺️ Mapa de Redes

| Red | Subred | Propósito |
|-----|--------|-----------|
| LAN Core | 10.0.1.0/24 | Infraestructura principal |
| BMC/IPMI | 10.99.0.0/24 | Gestión remota out-of-band N01–N06 |
| OPNsense Sync | 172.16.0.0/30 | Enlace dedicado HA entre H00 y H01 |
| Cluster Private | 192.168.10.0/24 | Red privada NIC2 — backup inter-server |
| ISP Uplink | 192.168.100.0/24 | Uplink Entel |

---

## 🔥 Firewall HA — OPNsense 26.1.2

Dos nodos en Alta Disponibilidad usando **CARP**. El nodo master (H00, menor `advskew`) procesa todo el tráfico. En fallo, H01 asume la VIP `10.0.1.220` en segundos sin intervención manual.

```
ISP Entel (192.168.100.1)
         │
    ┌────┴────┐
   H00       H01
10.0.1.1  10.0.1.2
   └──sync────┘
   172.16.0.0/30
         │
   CARP VIP: 10.0.1.220
         │
    Switch Core
```

| Nodo | Rol | IP LAN | IP Sync |
|------|-----|--------|---------|
| H00 | Master | 10.0.1.1 | 172.16.0.1/30 |
| H01 | Backup | 10.0.1.2 | 172.16.0.2/30 |
| CARP VIP | Gateway activo | 10.0.1.220 | — |

**pfSync** mantiene tablas de estado sincronizadas — las conexiones activas sobreviven un failover.

**Servicios:** Gateway, VPN (WireGuard/OpenVPN), NAT, DHCP server, reglas de firewall.

---

## 🌐 DNS — Redundancia completa

### H02 — DNS Primario (Armbian SBC — 10.0.1.3)

- **PiHole:** Filtrado de publicidad y dominios maliciosos a nivel de red
- **Unbound:** Resolver recursivo local — sin depender de DNS externos
- **Dominio interno:** `naucy.xyz`

> ⚠️ H02 es una SBC Armbian — hardware liviano. El DNS secundario en N07 garantiza resolución si H02 cae.

### N07 — DNS Secundario (LXC Debian — 10.0.1.70)

- **Unbound:** Instancia mínima de respaldo (~50 MB RAM)
- Sin PiHole — la disponibilidad prima sobre el filtrado en el secundario
- Todos los nodos configuran ambos servidores DNS: `10.0.1.3` (primario) y `10.0.1.70` (secundario)

---

## 🔒 BMC / IPMI — Gestión Out-of-Band

Red dedicada `10.99.0.0/24` para acceso remoto a servidores independiente del estado del OS. Permite reinicio, reinstalación y diagnóstico con el nodo completamente caído.

| Nodo | IP BMC |
|------|--------|
| N01 | 10.99.0.10 |
| N02 | 10.99.0.20 |
| N03 | 10.99.0.30 |
| N04 | 10.99.0.40 |
| N05 | 10.99.0.50 |
| N06 | 10.99.0.60 |

Acceso desde IRIS (IdeaPad 3) o cualquier equipo con ruta a `10.99.0.0/24`.

---

## 🔗 Red Privada Inter-Server (NIC2 — Cluster Private)

Todos los servidores (N01–N08) tienen una segunda NIC conectada a `192.168.10.0/24` mediante 3 routers en cascada — solución de reciclaje sin switch dedicado.

**Usos:**
- Heartbeat Corosync (Proxmox HA) aislado del LAN principal
- Canal de emergencia si el switch core falla
- Acceso de emergencia desde IRIS vía WiFi (`192.168.1.x`)

> Totalmente modificable o eliminable con un switch dedicado en el futuro.

---

## 📋 DHCP Reservations

Ver [`routers/dhcp-reservations.md`](routers/dhcp-reservations.md).

---

## 🚀 Guía de Despliegue

### 1. OPNsense H00 — Master

```bash
# Instalar OPNsense 26.1.2
# Interfaces:
#   WAN: uplink ISP
#   LAN: 10.0.1.1/24
#   SYNC: 172.16.0.1/30

# HA: System > High Availability > Settings
# Sync to IP: 172.16.0.2
# Sync: Firewall Rules, DHCP, VPN, Certificates
```

### 2. OPNsense H01 — Backup

```bash
# Misma versión de OPNsense
# LAN: 10.0.1.2/24
# SYNC: 172.16.0.2/30
# pfSync recibe configuración de H00
```

### 3. CARP VIP

```bash
# H00: Interfaces > Virtual IPs > Add
# Type: CARP | Interface: LAN
# IP: 10.0.1.220/24 | VHID: 1
# advskew H00: 0 (master) | H01: 100 (backup)
```

### 4. DNS H02 — Armbian SBC

```bash
# PiHole
curl -sSL https://install.pi-hole.net | bash

# Unbound
apt install unbound -y
# Configurar PiHole: Custom upstream → 127.0.0.1#5335
```

### 5. DNS Secundario N07 — LXC

```bash
# LXC Debian mínimo en Proxmox (512 MB RAM)
apt install unbound -y
# Configurar como resolver recursivo en 10.0.1.70
```
