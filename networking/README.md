# 🌐 Network Inventory & DHCP Reservations

Este documento centraliza el direccionamiento estático y las reservas de red del **HummerLab**. La red principal opera bajo el segmento `10.0.1.0/24`, gestionada por un clúster de **OPNsense en Alta Disponibilidad (HA)**.

---

## 📡 Infraestructura de Red (Gateways)


| Dispositivo | FQDN | IP Estática | Rol |
| :--- | :--- | :--- | :--- |
| **CARP VIP** | gateway.naucy.xyz | `10.0.1.1` | Puerta de enlace virtual |
| **H00** | h00.naucy.xyz | `10.0.1.2` | Firewall Master |
| **H01** | h01.naucy.xyz | `10.0.1.3` | Firewall Backup |
| **H02** | dns.naucy.xyz | `10.0.1.5` | DNS (PiHole/Unbound) |

---

## 🖥️ Mapeo de Nodos Físicos (DHCP Static Leases)

> **Nota:** Configurar estas reservas en OPNsense (`Services > DHCPv4 > [Interface]`) para asegurar la persistencia.


| Nodo | MAC Address (Ejemplo) | IP Estática | OS / Función |
| :--- | :--- | :--- | :--- |
| **N01** | AA:BB:CC:00:00:11 | `10.0.1.11` | Talos Master 1 |
| **N02** | AA:BB:CC:00:00:12 | `10.0.1.12` | Talos Master 2 |
| **N03** | AA:BB:CC:00:00:13 | `10.0.1.13` | Talos Worker 1 (IA) |
| **N04** | AA:BB:CC:00:00:14 | `10.0.1.14` | Talos Worker 2 (IA) |
| **N05** | AA:BB:CC:00:00:15 | `10.0.1.15` | Proxmox Node 1 |
| **N06** | AA:BB:CC:00:00:16 | `10.0.1.16` | Proxmox Node 2 |
| **N07** | AA:BB:CC:00:00:17 | `10.0.1.17` | Monitoring & QDevice |
| **N08** | AA:BB:CC:00:00:18 | `10.0.1.18` | Backup Server (PBS) |

---

## ⚖️ Direcciones Virtuales (VIPs) & Balanceo

Direcciones gestionadas mediante **Keepalived/HAProxy** para servicios de alta disponibilidad.


| Servicio | Virtual IP | FQDN | Nodo Destino |
| :--- | :--- | :--- | :--- |
| **Omni Manager** | `10.0.1.50` | omni.naucy.xyz | Proxmox Cluster |
| **K8s API** | `10.0.1.60` | k8s.naucy.xyz | Talos Masters |
| **DB Master** | `10.0.1.100` | db.naucy.xyz | Postgres (Patroni) |
| **Vector DB** | `10.0.1.110` | vector.naucy.xyz | Qdrant Cluster |
| **HomeAssistant** | `10.0.1.120` | ha.naucy.xyz | Proxmox HA |
| **Lab Entry** | `10.0.1.200` | lab.naucy.xyz | Ingress Controller |

---

## 🔒 Notas de Seguridad

*   **Aislamiento de Gestión:** Las interfaces de administración de Proxmox y Talos deben estar restringidas a la red de gestión.
*   **Inter-VLAN Routing:** *(Pendiente de implementar)* El tráfico de los modelos de IA hacia las bases de datos debe ser priorizado.
*   **DNS Local:** Todas las peticiones deben ser resueltas por `10.0.1.5` para garantizar el acceso mediante FQDN dentro del laboratorio.
