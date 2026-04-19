# 🛠️ HummerLab-Infrastructure

Arquitectura Self-Hosted de grado profesional para IA (RAG), Domótica (MCP) e Infraestructura Crítica.

## 🏗️ Topología del Sistema
- **Compute Layer:** 4 Nodos Talos OS (N01-N04) gestionados por Omni Local.
- **Data Layer:** 2 Nodos Proxmox VE (N05-N06) en clúster HA con QDevice en N07.
- **Security:** OPNsense HA (H00-H01) con balanceo de carga.

## 📡 Networking (Internal Domain: naucy.xyz)

| Host | Service | IP |
| :--- | :--- | :--- |
| **Talos API** | K8s Endpoint | 192.168.1.10 (VIP) |
| **Database** | Postgres/Qdrant | 192.168.1.100 (VIP) |
| **HomeAssistant**| MCP Controller | 192.168.1.101 (VIP) |

## 🚀 Roadmap
1. [ ] Configuración de OPNsense HA y VLANs.
2. [ ] Despliegue de Omni Manager en N06.
3. [ ] Aprovisionamiento de Talos Cluster (N01-N04).
4. [ ] Implementación de PostgreSQL con Patroni y PgVector.
