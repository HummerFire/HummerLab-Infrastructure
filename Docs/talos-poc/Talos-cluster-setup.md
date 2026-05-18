# Configuración del clúster Talos Linux (POC)

Este documento detalla el procedimiento seguido para crear un clúster de **Talos Linux** con 2 nodos (1 control‑plane, 1 worker) sobre VirtualBox, como parte de la prueba de concepto.

## Entorno de la prueba

| Elemento               | Detalle                                      |
|------------------------|----------------------------------------------|
| Hipervisor             | VirtualBox (modo puente)                     |
| Nodo control‑plane     | `t00` - IP `192.168.100.200/24`              |
| Nodo worker            | `t01` - IP `192.168.100.201/24`              |
| Máquina de gestión     | `mgmt` - Debian 13, IP `192.168.100.28/24`   |
| Versión de Talos       | v1.13.2                                      |
| Versión de Kubernetes  | v1.36.0 (la incluida en Talos v1.13.2)       |

## Requisitos previos

- VirtualBox instalado en el anfitrión.
- Imagen ISO de Talos Linux descargada (desde `https://github.com/siderolabs/talos/releases`).
- Dos máquinas virtuales creadas con:
  - 2 vCPUs, 2 GB RAM.
  - Red en modo **puente (Bridged Adapter)**.
  - Arranque desde la ISO.
- Una tercera VM con Debian 13 (o cualquier distribución Linux) que actuará como estación de control (`mgmt`), con acceso de red a las IPs de los nodos Talos.

## Paso 1: Creacion y configuración de nodo de administración.

1. Iniciar VM mgmt con ISO Debian 13.
   Se crea VM con ISO Debian 13, la cual se asigna hostname "mgmt", la cual será la que será el nexo con el cluster TALOS. En este caso se asigno IP estática:
   - `mgmt`: `192.168.100.28/24`
  
2. Instalación de Talos Linux CLI
   Puedes instalar automáticamente la versión correcta de talosctl para tu sistema operativo y arquitectura mediante un script de instalación.

```bash
curl -sL https://talos.dev/install | sh
```

3. Instalar y configurar kubectl
   Descarga la última versión con el comando:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```
  Ahora, se debe instalar kubectl, con el siguiente comando:

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
  Para verificar que la versión que has instalado esté actualizada:

```bash
kubectl version --client
```


## Paso 2: Arranque de los nodos Talos e identificación de IPs

1. Iniciar cada VM Talos con la ISO.  
   Talos arrancará en modo **maintenance** y mostrará su IP asignada por DHCP (o la configurada manualmente). En nuestro caso, usamos IPs estáticas configuradas en la red:
   - `t00`: `192.168.100.200/24`
   - `t01`: `192.168.100.201/24`

  Cree una variable para la dirección IP de sus nodos en la VM `mgmt`:
```bash
export CONTROL_PLANE_IP=192.168.100.200
export WORKER_IP=192.168.100.201
```

2. Desmonta la imagen ISO
   Esto evita que instales el programa accidentalmente en la unidad USB y facilita la selección del disco para la instalación.
   
3. Desde `mgmt`, verificar la conectividad y versión de Talos:

   ```bash
   talosctl version --insecure --nodes $CONTROL_PLANE_IP
   talosctl version --insecure --nodes $WORKER_IP
   ```
  Salida esperada: datos del cliente y del servidor Talos, indicando que el nodo está vivo.

4. Ejecute este comando para ver todos los discos disponibles en su nodo control plane:

   ```bash
   talosctl get disks --insecure --nodes $CONTROL_PLANE_IP
   ```

## Paso 3: Generar los archivos de configuración del clúster

1. Defina variables para el nombre de su clúster y el ID del disco.

   ```bash
   export CLUSTER_NAME=hummerlab
   export DISK_NAME=sda
   ```

2. Ejecute este comando para generar el archivo de configuración

   ```bash
   talosctl gen config $CLUSTER_NAME https://$CONTROL_PLANE_IP:6443 --install-disk /dev/$DISK_NAME
   ```
  Este comando generará tres archivos:
- **controlplane.yaml**: La configuración del plano de control.
- **worker.yaml**: La configuración de los nodos de trabajo.
- **talosconfig**: El archivo de configuración de talosctl, utilizado para conectarse al clúster y autenticar el acceso.


## Paso 4: Aplicar configuración al nodo control‑plane

```bash
talosctl apply-config --insecure --nodes $CONTROL_PLANE_IP --file controlplane.yaml
```
Recibiremos un ACK inmediato. El nodo comenzará a arrancar como control‑plane.


## Paso 5: Aplicar configuración al nodo worker

```bash
talosctl apply-config --insecure --nodes $WORKER_IP --file worker.yaml
```
No es necesario esperar a que el control‑plane termine; el worker se unirá automáticamente cuando el control‑plane esté listo.

## Paso 6: Configurar el cliente talosctl
Una vez que talosctl health sobre el control‑plane haya terminado con éxito, fusionamos el talosconfig y establecemos el endpoint:

```bash
talosctl --talosconfig=./talosconfig config endpoints $CONTROL_PLANE_IP
```
Verificamos que podemos comunicarnos de forma autenticada:

```bash
talosctl version
talosctl get members
```
Deberíamos ver los dos nodos (t00 y t01), con roles control-plane y worker respectivamente.

## Paso 7: Obtener el kubeconfig para kubectl

1. Integra tu nuevo clúster en tu configuración local de Kubernetes:
  
```bash
talosctl kubeconfig --nodes $CONTROL_PLANE_IP --talosconfig=./talosconfig
```

2. Compruebe el estado del clúster y verifique el registro del nodo.

   ```bash
   talosctl --nodes $CONTROL_PLANE_IP --talosconfig=./talosconfig health
   kubectl get nodes
   ```
   
Salida esperada:

```text
NAME   STATUS   ROLES           AGE    VERSION
t00    Ready    control-plane   176m   v1.36.0
t01    Ready    <none>          176m   v1.36.0
```

