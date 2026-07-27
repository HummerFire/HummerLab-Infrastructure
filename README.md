# 🔥 HummerLab Infrastructure

![License](https://img.shields.io/badge/License-GPL--2.0-blue.svg)
![Proxmox](https://img.shields.io/badge/Proxmox-VE%209-E57000?logo=proxmox&logoColor=white)
![Talos](https://img.shields.io/badge/Talos-OS-FF7300?logo=linux&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-K8s-326CE5?logo=kubernetes&logoColor=white)
![OPNsense](https://img.shields.io/badge/OPNsense-26.1.2%20HA-D94F00?logo=opnsense&logoColor=white)
![HomeAssistant](https://img.shields.io/badge/Home_Assistant-MCP-41BDF5?logo=home-assistant&logoColor=white)
![Status](https://img.shields.io/badge/Status-In_Progress-yellow)

> Arquitectura self-hosted de grado profesional para IA (RAG), Domótica (MCP) e Infraestructura Crítica. Control total, sin dependencias de nube, sin suscripciones. Construido sobre hardware reciclado en un rack de 32U.

---

## 📋 Tabla de Contenidos

- [¿Por qué este proyecto?](#-por-qué-este-proyecto)
- [Topología del Sistema](#-topología-del-sistema)
- [Inventario de Hardware](#-inventario-de-hardware)
- [Stack Tecnológico](#-stack-tecnológico)
- [Redes](#-redes)
- [Estructura del Repositorio](#-estructura-del-repositorio)
- [Roadmap](#-roadmap)

---

## 💡 ¿Por qué este proyecto?

La mayoría de las implementaciones de IA dependen de APIs externas, servicios en la nube y costos recurrentes. HummerLab es una apuesta por la **soberanía tecnológica**: inferencia local, datos que no salen de la red, alta disponibilidad sin vendor lock-in.

Construido íntegramente con hardware reciclado sobre un rack de 32U, este laboratorio corre modelos LLM locales, gestiona domótica crítica y almacena datos vectoriales — con configuraciones reproducibles, documentadas y listas para producción.

---

## 🌐 Topología del Sistema

```mermaid
%%{init: {"flowchart": {"layout": "elk"}} }%%
flowchart TB
    %% ============================================================
    %% CLASES Y ESTILOS
    %% ============================================================
    classDef wan stroke:#2dd4bf,fill:#f0fdfa
    classDef gw stroke:#818cf8,fill:#eef2ff,stroke-width:3px
    classDef dns stroke:#a78bfa,fill:#f5f3ff
    classDef ai stroke:#4ade80,fill:#f0fdf4
    classDef pve stroke:#22d3ee,fill:#ecfeff,stroke-width:3px
    classDef mgmt stroke:#fb923c,fill:#fff7ed
    classDef net stroke:#facc15,fill:#fefce8
    classDef redundant stroke:#fb7185,fill:#fff1f2,stroke-dasharray:5 3
    classDef vpn stroke:#34d399,fill:#ecfdf5,stroke-width:2px
    classDef cloud stroke:#60a5fa,fill:#eff6ff
    classDef isp stroke:#f97316,fill:#fff7ed,stroke-width:2px
    classDef legend stroke:#cbd5e1,fill:#f8fafc,stroke-dasharray:2 2

    %% ============================================================
    %% 1. INTERNET — DOS ISP
    %% ============================================================
    subgraph INTERNET["🌍 Internet — Dual ISP"]
        direction LR
        ISP1[("Entel<br>Fibra Óptica")]:::isp
        ISP2[("Movistar<br>Fibra Óptica")]:::isp
        INTERNET_GLOBAL(((Internet))):::wan
        INTERNET_GLOBAL --> ISP1
        INTERNET_GLOBAL --> ISP2
    end

    %% ============================================================
    %% 2. NUBE PÚBLICA — VPS HA (WireGuard)
    %% ============================================================
    subgraph CLOUD["☁️ Nube Pública — VPS HA"]
        direction LR
        VPS1["VPS001<br>Web + IREDMAIL + NGINX + WIREGUARD"]:::cloud
        VPS2["VPS002<br>DNS secundario + VPN backup + MX"]:::cloud
        VIP_CLOUD{{"VIP Pública<br>(WireGuard Endpoint)"}}:::redundant
        VPS1 <-->|"Sync / Heartbeat"| VPS2
    end

    %% ============================================================
    %% 3. HOMELAB — HummerLab
    %% ============================================================
    subgraph LAB["🏠 HummerLab — Selfhost Enviroment — naucy.lol"]
        direction TB

        %% ============================================================
        %% 3a. CLUSTER VPN (Pacemaker/PCS)
        %% ============================================================
        subgraph VPN_CLUSTER["🔒 Cluster VPN (Pacemaker/PCS)"]
            direction LR
            VPN1["VM VPN 1<br>Terminación WG"]:::vpn
            VPN2["VM VPN 2<br>Terminación WG (Backup)"]:::vpn
            VIP_VPN{{"VIP VPN<br>(WireGuard Termination)"}}:::redundant
            VPN1 <-->|"Corosync / Sync"| VPN2
        end

        %% ============================================================
        %% 3b. FIREWALL OPNsense HA (CARP) — Dual WAN via VLANs
        %% ============================================================
        subgraph GW["Gateway HA — OPNsense 26.1.2"]
            direction LR
            H00["H00 Master<br>10.0.1.1<br>WAN: eth0 (VLAN 100/200)<br>LAN: eth1<br>Sync: eth2"]:::gw
            H01["H01 Backup<br>10.0.1.2<br>WAN: eth0 (VLAN 100/200)<br>LAN: eth1<br>Sync: eth2"]:::gw
            SYNC["Sync Network<br>172.16.0.0/30"]:::net
            CARPVIP{{"CARP VIP<br>10.0.1.220"}}:::redundant

            H00 <-->|"pfsync / CARP"| H01
            H00 --- SYNC
            H01 --- SYNC
        end

        %% ============================================================
        %% 3c. INFRAESTRUCTURA INTERNA
        %% ============================================================
        subgraph DNS["DNS"]
            H02["H02 Armbian SBC<br>PiHole + Unbound<br>10.0.1.3"]:::dns
        end

        subgraph AI["AI Cluster — Talos OS + K8s"]
            direction LR
            N01["N01 CP1<br>10.0.1.10"]:::ai
            N02["N02 CP2<br>10.0.1.20"]:::ai
            N03["N03 CP3<br>10.0.1.30"]:::ai            
        end

        subgraph PVE["Proxmox HA Cluster"]
            direction LR
			N04["N04<br>10.0.1.40"]:::pve
            N05["N05<br>10.0.1.50"]:::pve
            N06["N06<br>10.0.1.60"]:::pve
        end

        subgraph MGMT["Management"]
            CANDY["CANDY Monitoring + DNS 2°<br>10.0.1.4"]:::mgmt
            IRIS["IRIS IdeaPad 3<br>Win10 MGMT<br>WiFi: 192.168.1.100"]:::mgmt
        end

        SWITCH["Switch Core<br>(VLAN aware)"]:::net
        BMC_NET["BMC/IPMI Network<br>10.99.0.0/24"]:::net

        %% ============================================================
        %% 4. CONEXIONES FÍSICAS Y LÓGICAS
        %% ============================================================

        %% 4a. ISP → OPNsense (Dual WAN sobre VLANs en eth0)
        ISP1 -->|"VLAN 100"| H00
        ISP1 -->|"VLAN 100"| H01
        ISP2 -->|"VLAN 200"| H00
        ISP2 -->|"VLAN 200"| H01

        %% 4b. Túnel WireGuard (Nube → VPN Cluster)
        VIP_CLOUD ===|"🔐 Túnel WireGuard<br>(UDP 51820)"| VIP_VPN

        %% 4c. VPN Cluster → Firewall
        VIP_VPN -->|"Tráfico entrante"| CARPVIP
        CARPVIP -->|"Anuncio CARP (Activo)"| H00
        CARPVIP -.->|"Standby"| H01

        %% 4d. Firewall → Switch Core (solo Master activo)
        H00 -->|"Ruteo + Firewalling"| SWITCH
        H01 -.->|"Standby"| SWITCH

        %% 4e. Switch → Servicios Internos
        SWITCH --- H02
        SWITCH --- AI
        SWITCH --- PVE
        SWITCH --- MGMT

        %% 4f. Tráfico de retorno (respuesta)
        SWITCH -->|"Respuesta"| H00
        H00 -->|"Respuesta"| CARPVIP
        CARPVIP -->|"Respuesta"| VIP_VPN
        VIP_VPN ===|"🔐 Respuesta (WG)"| VIP_CLOUD

        %% 4g. Gestión y monitoreo (SNMP / SSH)
        CANDY -.->|"SNMP / Monitoring"| H00
        CANDY -.->|"SNMP / Monitoring"| H01
		CANDY -.->|"SNMP / Monitoring"| H02
        CANDY -.->|"SNMP / Monitoring"| N01
        CANDY -.->|"SNMP / Monitoring"| N02
        CANDY -.->|"SNMP / Monitoring"| N03
        CANDY -.->|"SNMP / Monitoring"| N04
        CANDY -.->|"SNMP / Monitoring"| N05
        CANDY -.->|"SNMP / Monitoring"| N06

        %% 4h. BMC / IPMI (Out-of-band management)
        N01 --- BMC_NET
        N02 --- BMC_NET
        N03 --- BMC_NET
        N04 --- BMC_NET
        N05 --- BMC_NET
        N06 --- BMC_NET
        IRIS -.->|"IPMI / BMC"| BMC_NET
    end

    %% ============================================================
    %% 5. LEYENDA
    %% ============================================================
    subgraph LEGEND["📘 Leyenda"]
        direction TB
        L1["⬜ Línea sólida = enlace físico / datos"]:::legend
        L2["⬜ Línea punteada = enlace lógico / gestión / standby"]:::legend
        L3["⬜ Línea gruesa doble = Túnel WireGuard (encriptado)"]:::legend
        L4["⬜ Borde punteado = IP virtual compartida (VIP)"]:::legend
        L5["⬜ Borde más grueso = nodo redundante / HA"]:::legend
        L6["⬜ Línea segmentada con texto = VLAN (trunk)"]:::legend
        L7["⬜ Las IPs BMC (10.99.0.x) están en red de gestión separada"]:::legend
    end

    LAB --> LEGEND
```

---

## 🖥️ Inventario de Hardware

### Baremetal — Rack (32U)

| Nodo | Rol | CPU | RAM | Storage | OS | IP LAN | BMC |
|------|-----|-----|-----|---------|-----|--------|-----|
| H00 | OPNsense Master | 2c | 4 GB | 80 GiB | OPNsense 26.1.2 | 10.0.1.1 | — |
| H01 | OPNsense Backup | 2c | 4 GB | 40 GiB | OPNsense 26.1.2 | 10.0.1.2 | — |
| H02 | DNS / PiHole / Unbound | 2c | 1 GB | 50 GiB | Armbian | 10.0.1.3 | — |
| N01 | Talos — Control Plane 1 | 8c | 32 GB | 4× 500 GiB | Talos OS | 10.0.1.10 | 10.99.0.10 |
| N02 | Talos — Control Plane 2 | 8c | 32 GB | 3× 500 GiB + 1 TiB | Talos OS | 10.0.1.20 | 10.99.0.20 |
| N03 | Talos — Worker IA | 8c | 32 GB | 2× 500 GiB + 2× 1 TiB | Talos OS | 10.0.1.30 | 10.99.0.30 |
| N04 | Talos — Worker IA | 8c | 32 GB | 2× 500 GiB + 2× 1 TiB | Talos OS | 10.0.1.40 | 10.99.0.40 |
| N05 | Proxmox Node 1 | 8c | 32 GB | 2× 500 GiB + 2× 1 TiB | Proxmox VE 9 | 10.0.1.50 | 10.99.0.50 |
| N06 | Proxmox Node 2 | 8c | 32 GB | 2× 320 GiB + 2× 1 TiB | Proxmox VE 9 | 10.0.1.60 | 10.99.0.60 |
| N07 | Monitoring + QDevice + DNS 2° | 4c | 6 GB | 250 GiB SSD | Debian 12 | 10.0.1.70 | — |
| N08 | PBS Backup | 2c | 2 GB | 4× SATA + 4× IDE | Debian 12 | 10.0.1.80 | — |

### Fuera de Rack

| Equipo | Rol | CPU | RAM | Storage | OS | IP |
|--------|-----|-----|-----|---------|----|----|
| IRIS (IdeaPad 3) | MGMT Personal / Backup | 4c | 20 GB | 500 GiB SSD + 1 TiB | Windows 10 | WiFi 192.168.1.100 |

### VPS Externos

| Nombre | Rol | CPU | RAM | Storage | OS |
|--------|-----|-----|-----|---------|-----|
| VPS001 | Web + Mail + DNS + VPN | 2c | 4 GB | 60 GiB | Ubuntu 24.04 |
| VPS002 | DNS secundario + VPN backup + MX | 4c | 8 GB | 140 GiB SSD | Ubuntu 24.04 |

---

## 🏗️ Stack Tecnológico

| Capa | Tecnología | Rol |
|------|-----------|-----|
| **Firewall/Gateway** | OPNsense 26.1.2 HA (CARP) | Entrada redundante, VPN, segmentación |
| **DNS primario** | PiHole + Unbound (H02 — Armbian SBC) | Filtrado + resolución recursiva |
| **DNS secundario** | Unbound LXC (N07) | Redundancia ante caída de H02 |
| **Compute IA** | Talos OS + Kubernetes | Cómputo inmutable para inferencia LLM |
| **Orquestación K8s** | Omni — Sidero Labs | Gestión del ciclo de vida Talos |
| **Inferencia LLM** | Ollama / LocalAI | Modelos locales sin API externa |
| **Virtualización** | Proxmox VE 9 HA | VMs y LXCs con alta disponibilidad |
| **Base de datos** | PostgreSQL + PgVector | Memoria largo plazo para RAG |
| **Vector Search** | Qdrant HA | Búsqueda semántica replicada |
| **HA DB** | Patroni + Etcd | Failover automático PostgreSQL |
| **Cache** | Redis + Sentinel | Cache HA para aplicaciones |
| **Load Balancer** | HAProxy + Keepalived + PgBouncer | VIPs y balanceo interno |
| **Domótica** | Home Assistant OS | Controlador MCP — Zigbee/Z-Wave |
| **Backup** | Proxmox Backup Server (N08) | Deduplicación + recuperación |
| **Quórum PVE** | Corosync QNetd (N07) | Árbitro externo anti split-brain |
| **Monitoreo** | VictoriaMetrics + Grafana + Loki | Observabilidad full-stack |
| **Gestión remota** | BMC/IPMI (10.99.0.x) | Acceso out-of-band a N01–N06 |

---

## 📡 Redes

**Dominio:** `naucy.xyz` | **Subred principal:** `10.0.1.0/24`

| Red | Subred | Propósito |
|-----|--------|-----------|
| LAN Core | 10.0.1.0/24 | Infraestructura principal |
| BMC/IPMI | 10.99.0.0/24 | Gestión remota out-of-band N01–N06 |
| OPNsense Sync | 172.16.0.0/30 | Enlace dedicado HA H00↔H01 |
| Cluster Private | 192.168.10.0/24 | Red privada NIC2 — backup inter-server |
| ISP | 192.168.100.0/24 | Uplink Entel |

### VIPs

| VIP | Servicio | IP |
|-----|---------|-----|
| CARP OPNsense | Gateway activo | 10.0.1.220 |
| Talos API | K8s Endpoint | 10.0.1.200 |
| DB VIP | PostgreSQL / Qdrant | 10.0.1.221 |
| HA VIP | Home Assistant | 10.0.1.222 |

---

## 📁 Estructura del Repositorio

```
HummerLab-Infrastructure/
├── Docs/               # ADRs, runbooks y referencias técnicas
├── Kubernetes/         # Manifests y configuraciones K8s
├── Proxmox/            # Configuración clúster PVE HA
├── Talos/              # Configs Talos y aprovisionamiento Omni
├── Topology/           # Diagramas de red y arquitectura
└── networking/         # OPNsense, DHCP, BMC, redes
```

---

## 🚀 Roadmap

- [x] Diseño de arquitectura y topología
- [x] Documentación capa Proxmox HA
- [x] Diagrama de red completo
- [x] Inventario de hardware y redes
- [ ] **Fase 1** — OPNsense HA + CARP + VPN
- [ ] **Fase 2** — Aprovisionamiento Talos Cluster (Omni)
- [ ] **Fase 3** — Proxmox HA + QDevice + PBS
- [ ] **Fase 4** — PostgreSQL HA (Patroni + PgVector + Qdrant + Redis)
- [ ] **Fase 5** — Stack IA (Ollama + RAG pipeline + Open WebUI)
- [ ] **Fase 6** — Home Assistant + integración MCP
- [ ] **Fase 7** — Observabilidad (VictoriaMetrics + Grafana + Loki)
- [ ] **Fase 8** — CI/CD y automatización del repo

---

**Infraestructura de grado profesional no requiere presupuesto enterprise — requiere criterio técnico, reciclaje inteligente y documentación seria.**
*GPL-2.0 — Úsalo, mejóralo, compártelo.*

---

## 📬 ¿Te sirve? ¿Tienes preguntas?

- 🐛 Reporta issues si algo no cuadra.
- 💬 Conéctame en [LinkedIn](https://www.linkedin.com/in/patricio-nunez-rgua/) si quieres charlar infra, IA o homelabs.

> *"El mejor código es el que otros pueden entender, mejorar y hacer suyo."*
