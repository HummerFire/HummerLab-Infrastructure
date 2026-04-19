## 🛡️ Data & Infrastructure Layer (Proxmox VE Cluster)

Esta sección del HummerLab constituye el "Estado" y la "Persistencia" del ecosistema. Mientras que el clúster de Talos OS se encarga del cómputo efímero e inferencia de IA, este segmento en Proxmox garantiza que los datos, la domótica y los servicios críticos sean resilientes, respaldados y altamente disponibles.

## 🏗️ Arquitectura del Segmento

El clúster está compuesto por dos nodos principales de virtualización y un árbitro externo para garantizar el quórum, eliminando cualquier punto único de falla (SPOF).

- **Nodos de Cómputo/Storage (N05 & N06):** Servidores Proxmox VE 9 configurados en clúster HA.
- **QDevice (N07):** Nodo Debian 12 externo que actúa como tercer voto (Corosync QNetd) para evitar escenarios de Split-Brain.
- **Backup Server (N08):** Proxmox Backup Server (PBS) dedicado para deduplicación y recuperación ante desastres.

## 🚀 Servicios Alojados

En este clúster residen las máquinas virtuales (VMs) y contenedores (LXCs) que requieren estado persistente:

- **Omni Manager (Control Plane de Talos):** El cerebro que gestiona el ciclo de vida de los 4 nodos de IA.
- **Database Cluster (PostgreSQL + PgVector):** Gestionado con Patroni para alta disponibilidad, almacenando la memoria de largo plazo del RAG.
- **Vector Search (Qdrant HA):** Motor de búsqueda semántica replicado.
- **Home Automation (Home Assistant OS):** Instancia crítica para la domótica (MCP), con passthrough de hardware para protocolos Zigbee/Z-Wave.
- **Traffic Management:** HAProxy + Keepalived gestionando las VIPs (Virtual IPs) de la red 10.0.1.x.

## 🛠️ Configuraciones Relevantes

- **High Availability (HA):** Configurada con grupos de recursos para que, en caso de fallo de un nodo, las VMs críticas migren automáticamente al nodo sobreviviente.
- **Replicación ZFS:** (Opcional/Configurado) Sincronización de almacenamiento local entre N05 y N06 cada 15 minutos para minimizar el RPO (Recovery Point Objective).
- **Networking:** Implementación de Linux Bridges para segmentación de tráfico mediante VLANs gestionadas por los firewalls OPNsense.

## 📋 Guía de Despliegue Rápido

1. [ ]    Instalación de PVE en N05 y N06.
2. [ ]    Creación del clúster:
   pvecm create HummerCluster
3. [ ]    Unión de nodos:
   pvecm add [IP-N05]
4. [ ]    Configuración del QDevice en N07:
   pvecm qdevice setup 10.0.1.17
5. [ ]    Configuración de Storage NFS/Shared para el repositorio de ISOS y Backups.

**Este segmento es parte del proyecto HummerLab, una iniciativa para democratizar infraestructuras de IA soberanas y domótica avanzada.**
