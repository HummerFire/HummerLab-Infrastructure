# 🤖 Talos OS Cluster — HummerLab

Aprovisionamiento y configuración del clúster Kubernetes sobre **Talos OS**, gestionado con **Omni** (Sidero Labs).

---

## ¿Por qué Talos OS?

OS diseñado exclusivamente para Kubernetes. Sin SSH, sin shell, sin paquetes extra. **Inmutable, declarativo y seguro por diseño** — ideal para nodos de inferencia IA donde la estabilidad y reproducibilidad son críticas.

---

## 🏗️ Arquitectura del Clúster

| Nodo | Rol | IP LAN | BMC | CPU | RAM | Storage |
|------|-----|--------|-----|-----|-----|---------|
| N01 | Control Plane 1 | 10.0.1.10 | 10.99.0.10 | 8c | 32 GB | 4× 500 GiB |
| N02 | Control Plane 2 | 10.0.1.20 | 10.99.0.20 | 8c | 32 GB | 3× 500 GiB + 1 TiB |
| N03 | Worker — IA | 10.0.1.30 | 10.99.0.30 | 8c | 32 GB | 2× 500 GiB + 2× 1 TiB |
| N04 | Worker — IA | 10.0.1.40 | 10.99.0.40 | 8c | 32 GB | 2× 500 GiB + 2× 1 TiB |
| **Talos API VIP** | K8s Endpoint | **10.0.1.200** | — | — | — | — |

2 control planes garantizan quórum sin SPOF. 2 workers dedicados a inferencia con máximo storage disponible.

> **BMC/IPMI disponible en todos los nodos** — acceso out-of-band para reinstalación y diagnóstico remoto desde `10.99.0.x`.

---

## 📁 Estructura

```
Talos/
├── README.md
├── controlplane.yaml       # Config nodos master (Omni)
├── worker.yaml             # Config nodos worker (Omni)
├── omni-cluster.yaml       # Definición del clúster en Omni
└── patches/
    └── storage.yaml        # Patch para configuración de discos
```

---

## 🛠️ Gestión con Omni

**Omni** corre como VM en el clúster Proxmox (N05/N06) y permite:
- Aprovisionamiento declarativo de nodos Talos
- Upgrades del clúster sin downtime
- Dashboard web para gestión del ciclo de vida
- Generación de `talosctl` kubeconfig

```bash
# Acceso interno
# https://omni.naucy.xyz
```

---

## 🚀 Guía de Aprovisionamiento

### Prerrequisitos
- Red `10.0.1.0/24` operativa (OPNsense HA funcionando)
- DHCP reservations configuradas para N01–N04
- Omni Manager corriendo en Proxmox
- ISO Talos OS para bare metal (USB booteable)

### 1. Bootear nodos con ISO Talos

```bash
# Bootear N01–N04 con ISO de Talos
# Los nodos aparecen automáticamente en el dashboard de Omni
# como "not allocated" — identificados por MAC address
```

### 2. Crear el clúster en Omni

```bash
omnictl cluster create HummerCluster \
  --control-plane-nodes 2 \
  --worker-nodes 2 \
  --kubernetes-version 1.31.0
```

### 3. Kubeconfig y verificación

```bash
omnictl kubeconfig > ~/.kube/config
kubectl get nodes -o wide

# Esperado:
# n01   Ready   control-plane   10.0.1.10
# n02   Ready   control-plane   10.0.1.20
# n03   Ready   worker          10.0.1.30
# n04   Ready   worker          10.0.1.40
```

### 4. Health check

```bash
talosctl --nodes 10.0.1.10,10.0.1.20 health
talosctl --nodes 10.0.1.30,10.0.1.40 health
```

---

## 🧠 Workloads de IA (Workers N03, N04)

| Servicio | Descripción | Namespace |
|---------|-------------|-----------|
| **Ollama** | Inferencia LLM local (texto, prompts, narrativa) | `ai` |
| **LocalAI** | API compatible OpenAI sobre modelos locales | `ai` |
| **Open WebUI** | Interfaz web para modelos | `ai` |

> Manifests de despliegue en `../Kubernetes/ai/`

---

## 📝 Notas de Hardware

- **CPU-only inference** — sin GPU por ahora. Los modelos corren en CPU (8 cores × 2 workers). Recomendado: modelos ≤ 13B para latencia aceptable.
- **Storage distribuido** — N03 y N04 tienen mayor capacidad de disco para almacenar modelos localmente.
- **Expansión GPU futura** — se contempla agregar GPUs vía riser PCIe cuando el presupuesto lo permita. NVIDIA RTX 3060 12GB (CUDA) es el target por ecosistema K8s.
