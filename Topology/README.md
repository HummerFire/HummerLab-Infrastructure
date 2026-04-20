# 🌐 HummerLab Network Topology

Este documento detalla la arquitectura de red del **HummerLab**, un ecosistema diseñado para la alta disponibilidad (HA), la soberanía de datos y el cómputo de Inteligencia Artificial. La red se organiza en una topología de estrella con un núcleo de red central y capas de servicios especializados.

## 🗺️ Mapa de Red ([Visualización](https://mermaid.ai/d/232fba26-2a5b-41c8-be71-ae3e3bbfff9c))

```mermaid
flowchart TB
 subgraph WAN["🌐 ISP"]
        ISP["Entel<br>192.168.100.1"]
  end
 subgraph Gateway["📡 Gateway Infrastructure"]
    direction LR
        H00["H00 Master<br>10.0.1.2"]
        H01["H01 Backup<br>10.0.1.3"]
        CARPVIP["CARP VIP<br>10.0.1.1"]
        H02["H02 DNS Server<br>10.0.1.5"]
  end
 subgraph Cluster["🤖 LLM Nodes (Talos OS)"]
    direction TB
        N01["N01 Master 1<br>10.0.1.11"]
        N02["N02 Master 2<br>10.0.1.12"]
        N03["N03 Worker 1 IA<br>10.0.1.13"]
        N04["N04 Worker 2 IA<br>10.0.1.14"]
  end
 subgraph ClusterB["🖥️ Service Nodes (PVE HA)"]
    direction TB
        N05["N05 Node 1<br>10.0.1.15"]
        N06["N06 Node 2<br>10.0.1.16"]
  end
 subgraph ClusterC["🛠️ MGNT Nodes"]
    direction TB
        N07["N07 Monitoring<br>10.0.1.17"]
        N08["N08 PBS Backup<br>10.0.1.18"]
  end
 subgraph Network["<b style='font-size:22px'>🔥 HummerLab 🔥</b>"]
        WAN
        Gateway
        SW["🔌 Switch LAN (Core)"]
        Cluster
        ClusterB
        ClusterC
  end
    ISP --> H00 & H01
    H00 --- CARPVIP
    H01 --- CARPVIP
    CARPVIP --> H02
    H02 --> SW
    SW --- Cluster & ClusterB & ClusterC

     H00:::gateway
     H01:::gateway
     CARPVIP:::gateway
     H02:::gateway
     SW:::core
     N01:::talos
     N02:::talos
     N03:::talos
     N04:::talos
     N05:::pve
     N06:::pve
     N07:::mgnt
     N08:::mgnt
    classDef gateway fill:#fdd,stroke:#922,stroke-width:2px,color:#000
    classDef talos fill:#dcf,stroke:#428,stroke-width:2px,color:#000
    classDef pve fill:#dfd,stroke:#262,stroke-width:2px,color:#000
    classDef mgnt fill:#eee,stroke:#444,stroke-width:2px,color:#000
    classDef core fill:#fff,stroke:#000,stroke-width:4px,color:#000
    style Network color:#FF6D00,stroke-width:3px

```

## 🏗️ Desglose de Capas

### 📡 1. Infraestructura de Gateway (HA)
El punto de entrada está protegido por un clúster de firewalls en **Alta Disponibilidad (HA)** utilizando el protocolo **CARP**.
* **Redundancia:** Si el nodo `H00` falla, `H01` asume el tráfico de forma transparente mediante la **CARP VIP** (`10.0.1.1`).
* **Servicio DNS:** El nodo `H02` centraliza la resolución de nombres local y el filtrado de publicidad/amenazas mediante **PiHole/Unbound**.

### 🤖 2. LLM Nodes (Cómputo Inmutable)
Segmento dedicado a la inteligencia artificial ejecutando **Talos OS**, un sistema operativo endurecido y diseñado específicamente para Kubernetes.
* **Control Plane:** Dos masters gestionan la orquestación y el estado del clúster.
* **Inferencia:** Dos workers equipados para el procesamiento de lenguaje natural y ejecución de modelos locales (**Ollama/LocalAI**).

### 🖥️ 3. Service Nodes (Persistencia)
Nodos corriendo **Proxmox VE (PVE)** en configuración de alta disponibilidad para servicios críticos.
* **Estado:** Alojan bases de datos relacionales y vectoriales (**PostgreSQL/Qdrant**) junto a la instancia de **Home Assistant**.
* **Resiliencia:** Configurados con replicación de almacenamiento para permitir migraciones en caliente (*live migration*) sin interrupción de servicio.

### 🛠️ 4. MGNT Nodes (Apoyo & Gestión)
Infraestructura vital para el mantenimiento, observabilidad e integridad del laboratorio.
* **N07 (Monitoring):** Centraliza métricas/logs y actúa como **QDevice** para asegurar el quórum legal en el clúster Proxmox.
* **N08 (Backup):** Servidor de respaldos dedicado (**Proxmox Backup Server**) que garantiza la recuperación ante desastres de todos los nodos del ecosistema.

---

> [!IMPORTANT]
> **Nota Técnica:** Todas las comunicaciones internas fluyen a través de un **Switch LAN Core** de baja latencia, optimizando el rendimiento entre los motores de inferencia IA y las bases de datos vectoriales.
