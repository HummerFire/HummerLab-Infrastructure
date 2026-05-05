# ☸️ Kubernetes — HummerLab

Manifests y configuraciones de workloads desplegados en el clúster Talos OS de HummerLab.

---

## 🏗️ Estructura

```
Kubernetes/
├── README.md
├── namespaces.yaml
├── ai/
│   ├── ollama.yaml             # Inferencia LLM local
│   ├── localai.yaml            # API compatible OpenAI
│   └── open-webui.yaml         # UI web para modelos
├── data/
│   ├── qdrant.yaml             # Vector DB HA
│   └── pg-operator.yaml        # Conexión a PostgreSQL externo
├── ingress/
│   └── ingress-nginx.yaml
├── monitoring/
│   ├── victoriametrics.yaml
│   ├── grafana.yaml
│   └── loki.yaml
└── storage/
    └── storageclass.yaml
```

---

## 🌐 Namespaces

| Namespace | Propósito |
|-----------|-----------|
| `ai` | Inferencia LLM (Ollama, LocalAI, Open WebUI) |
| `data` | Qdrant vector DB |
| `monitoring` | VictoriaMetrics, Grafana, Loki |
| `ingress-nginx` | Ingress controller |
| `home-automation` | Home Assistant (si aplica) |

---

## 🤖 Stack de IA

### Ollama — Inferencia LLM

```bash
kubectl apply -f ai/ollama.yaml

# Verificar
kubectl -n ai get pods
kubectl -n ai logs deploy/ollama

# API interna
curl http://ollama.naucy.xyz/api/generate \
  -d '{"model": "llama3", "prompt": "Hola mundo"}'
```

Modelos recomendados para CPU-only (8c × 2 workers):
- `llama3.2:3b` — rápido, buena calidad
- `mistral:7b` — equilibrio calidad/velocidad
- `phi3:mini` — muy liviano, bueno para pruebas

### LocalAI — Capa OpenAI compatible

```bash
kubectl apply -f ai/localai.yaml
# Endpoint: http://localai.naucy.xyz/v1
# Compatible con cualquier cliente OpenAI apuntando a este endpoint
```

---

## 🗄️ Stack de Datos

### Qdrant — Vector DB HA

```bash
kubectl apply -f data/qdrant.yaml
# REST API: http://qdrant.naucy.xyz:6333
# gRPC:     qdrant.naucy.xyz:6334
```

### PostgreSQL + PgVector

> PostgreSQL **no corre en K8s** — corre en el clúster Proxmox (N05/N06) gestionado por Patroni con VIP `10.0.1.221`. Los pods K8s se conectan al VIP externo. Esta decisión garantiza persistencia con respaldo ZFS nativo de Proxmox.

```bash
# Conexión desde pods K8s
# Host: 10.0.1.221
# Puerto: 5432 (via PgBouncer)
```

---

## 📊 Monitoreo

```bash
kubectl apply -f monitoring/

# Grafana: http://grafana.naucy.xyz
# VictoriaMetrics: http://vm.naucy.xyz:8428
# Loki: http://loki.naucy.xyz:3100
```

> Se usa **VictoriaMetrics** en lugar de Prometheus — menor consumo de RAM/CPU, compatible con PromQL, ideal para el hardware disponible en N07 (6 GB RAM).

---

## 🚀 Deploy Completo

```bash
git clone https://github.com/HummerFire/HummerLab-Infrastructure.git
cd HummerLab-Infrastructure/Kubernetes

# Orden de despliegue
kubectl apply -f namespaces.yaml
kubectl apply -f storage/
kubectl apply -f data/
kubectl apply -f ai/
kubectl apply -f monitoring/
kubectl apply -f ingress/
```
