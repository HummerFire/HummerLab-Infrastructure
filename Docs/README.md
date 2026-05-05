# 📚 Docs — HummerLab

Decisiones de arquitectura (ADRs), runbooks operacionales y referencias técnicas.

---

## 📁 Estructura

```
Docs/
├── README.md
├── adr/
│   ├── ADR-001-talos-vs-k3s.md
│   ├── ADR-002-proxmox-vs-vmware.md
│   ├── ADR-003-qdrant-vs-weaviate.md
│   ├── ADR-004-victoriametrics-vs-prometheus.md
│   └── ADR-005-opnsense-vs-pfsense.md
├── runbooks/
│   ├── failover-opnsense.md
│   ├── proxmox-node-recovery.md
│   └── talos-node-reboot.md
└── references/
    └── ip-inventory.md
```

---

## 🧠 Architecture Decision Records

### ADR-001: Talos OS vs K3s / RKE2

**Decisión:** Talos OS + Omni.

**Razones:** OS inmutable sin SSH ni shell — superficie de ataque mínima. Reproducible por diseño, sin config drift. Omni permite gestión centralizada y upgrades declarativos. Ideal para nodos de inferencia IA donde la estabilidad es crítica.

**Descartado:** K3s (demasiado simple para workloads IA HA), RKE2 (overhead mayor sin ventajas claras para este caso).

---

### ADR-002: Proxmox VE vs VMware ESXi

**Decisión:** Proxmox VE 9 HA.

**Razones:** Open source, sin licencias. QDevice para quórum sin tercer nodo completo. ZFS nativo con replicación entre nodos. PBS integrado. Comunidad activa.

**Descartado:** VMware ESXi (licenciamiento prohibitivo post-Broadcom), Harvester HCI (menos maduro para casos mixtos).

---

### ADR-003: Qdrant vs Weaviate vs Chroma

**Decisión:** Qdrant HA.

**Razones:** Escrito en Rust — mejor performance y uso de memoria. Soporte nativo de replicación y sharding. API REST + gRPC limpia. Filtros de payload eficientes para RAG híbrido.

**Descartado:** Weaviate (mayor overhead de recursos), Chroma (sin soporte HA real).

---

### ADR-004: VictoriaMetrics vs Prometheus

**Decisión:** VictoriaMetrics.

**Razones:** N07 tiene solo 6 GB RAM — VictoriaMetrics consume hasta 7× menos memoria que Prometheus con el mismo volumen de métricas. Compatible con PromQL y dashboards Grafana existentes. Sin cambios en los scrape configs.

**Descartado:** Prometheus (demasiado pesado para el hardware disponible en N07).

---

### ADR-005: OPNsense vs pfSense

**Decisión:** OPNsense 26.1.2.

**Razones:** UI más moderna y organizada. Actualizaciones más frecuentes. Plugin system más flexible. Mejor soporte para WireGuard nativo. Comunidad activa con roadmap claro.

**Descartado:** pfSense (divergencia del fork CE, modelo de negocio más restrictivo post-Netgate).

---

## 📋 Runbooks

### Failover manual OPNsense

```bash
# Si H00 falla y el CARP no conmuta automáticamente:
# En H01 — forzar toma de VIP
# Interfaces > Virtual IPs > bajar advskew a 0
# El tráfico comienza a fluir por H01 inmediatamente
```

### Recuperación de nodo Proxmox

```bash
# Verificar estado del clúster
pvecm status

# Si el nodo está offline pero el OS responde via BMC:
# Acceder via IPMI (10.99.0.x) y reiniciar
ipmitool -H 10.99.0.50 -U admin -P [pass] chassis power reset
```

### Reinicio seguro de nodo Talos

```bash
# Drenar el nodo antes de reiniciar
kubectl drain n03.naucy.xyz --ignore-daemonsets --delete-emptydir-data

# Reiniciar via talosctl
talosctl reboot --nodes 10.0.1.30

# Re-habilitar scheduling
kubectl uncordon n03.naucy.xyz
```

---

## 📡 Inventario de Red

Ver [`references/ip-inventory.md`](references/ip-inventory.md) para el inventario completo con IPs, MACs y roles de cada dispositivo.

---

> **Filosofía:** Documentar el *por qué*, no solo el *cómo*. Cualquier decisión que cueste más de 30 minutos revertir merece un ADR.
