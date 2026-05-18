# Prueba de concepto: Clúster Talos en VirtualBox

**Fecha:** Mayo 2026  
**Propósito:** Validar la instalación de Talos Linux, MetalLB y stack de monitoreo en un entorno controlado antes de proceder a la implementación definitiva del homelab.

## ⚠️ Naturaleza de este documento

Esta POC se centra exclusivamente en validar **Talos Linux como sistema operativo base** para Kubernetes, incluyendo la integración con MetalLB y un stack de monitoreo. No representa la configuración final del homelab, que usará estos conocimientos pero con automatización y diferentes parámetros.

- Los parámetros (IPs, número de nodos, versión de Talos) son meramente ilustrativos.
- El objetivo fue verificar la viabilidad técnica y documentar el procedimiento para futuras automatizaciones.
- Para la instalación real del homelab, se utilizarán scripts automatizados y configuraciones diferentes.

## Entorno de prueba

| Recurso               | Configuración                      |
|-----------------------|------------------------------------|
| Hipervisor            | VirtualBox (modo puente)           |
| Nodo control-plane    | Talos v1.13.2 - IP 192.168.100.200 |
| Nodo worker           | Talos v1.13.2 - IP 192.168.100.201 |
| Máquina de gestión    | Debian 13 - IP 192.168.100.28      |
| Red                   | LAN 192.168.100.0/24               |
| Rango MetalLB         | 192.168.100.210 - 192.168.100.220  |

## Resultados obtenidos

✅ Talos Linux instalado y configurado correctamente.  
✅ Clúster Kubernetes operativo con 2 nodos (1 CP, 1 worker).  
✅ MetalLB funcional asignando IPs del rango a servicios `LoadBalancer`.  
✅ Stack de monitoreo (Prometheus + Grafana) desplegado con Helm.  
✅ Acceso desde cualquier equipo de la LAN a:
  - Nginx de prueba → `http://192.168.100.210`
  - Grafana → `http://192.168.100.211` (usuario `admin`, contraseña `prom-operator`)
  - Prometheus → `http://192.168.100.212:9090`

## Comandos y procedimientos

Para el detalle paso a paso de cada componente, consultar los archivos en esta misma carpeta:

- [`Talos-cluster-setup.md`](./Talos-cluster-setup.md)
- [`metallb-test.md`](./metallb-test.md)
- [`monitoring-test.md`](./monitoring-test.md)

## Conclusiones y lecciones aprendidas

- El stack elegido (Talos + MetalLB + Prometheus/Grafana) es viable y se comportó según lo esperado.
- La instalación manual es factible pero requiere atención y varios pasos. **Es imprescindible automatizarla** para el homelab real.
- MetalLB funciona perfectamente en modo L2 con VirtualBox en modo puente.
- El stack de monitoreo con Helm es sencillo de desplegar, pero la exposición con `LoadBalancer` requiere parchear los servicios o usar valores personalizados.

## Próximos pasos para el homelab real

- [ ] Automatizar el despliegue con scripts (bash/Makefile) o Terraform.
- [ ] Definir IPs definitivas y rango de MetalLB adaptado a la red real.
- [ ] Integrar GitOps con Flux.
- [ ] Desplegar almacenamiento persistente (Longhorn o NFS).
- [ ] Incorporar aplicaciones de IA (Ollama, PgVector, Qdrant).

---

*Nota: Esta prueba no debe utilizarse como base para un entorno de producción. El homelab final tendrá requisitos de alta disponibilidad, seguridad y rendimiento diferentes.*
