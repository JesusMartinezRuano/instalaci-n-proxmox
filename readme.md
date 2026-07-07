Activar en la BIOS el soporte de 


Si quieres que **OpenSSH solo escuche en la IP del servidor**, puedes añadir la directiva `ListenAddress` en `sshd_config`.

Suponiendo que la IP del servidor es `192.168.1.10`, la receta completa sería:

```bash
# Editar la configuración
nano /etc/ssh/sshd_config
```

Añadir o modificar la línea:

```text
ListenAddress 192.168.1.10
```

Si prefieres hacerlo desde la línea de comandos:

```bash
SERVER_IP="192.168.1.10"

grep -q '^ListenAddress' /etc/ssh/sshd_config \
    && sed -i "s/^ListenAddress.*/ListenAddress ${SERVER_IP}/" /etc/ssh/sshd_config \
    || echo "ListenAddress ${SERVER_IP}" >> /etc/ssh/sshd_config
```

Comprobar que la configuración es correcta:

```bash
sshd -t
```

Reiniciar el servicio SSH:

```bash
systemctl restart ssh
```

Verificar que SSH escucha únicamente en la dirección configurada:

```bash
ss -tlnp | grep :22
```

La salida debería ser similar a:

```text
LISTEN 0 128 192.168.1.10:22 0.0.0.0:* users:(("sshd",pid=...,fd=3))
```

> **Importante:** antes de reiniciar `sshd`, asegúrate de que la IP indicada está realmente asignada al servidor. Si configuras una dirección incorrecta, podrías perder el acceso remoto por SSH.



Como hemos comprobado durante las pruebas, **no necesitas las reglas con `MARK`**. Basta con una redirección NAT del puerto 443 al 8006.

### 1. Eliminar cualquier regla anterior

```bash
iptables -t nat -F
iptables -t mangle -F
```

### 2. Redirección del tráfico entrante (443 → 8006)

```bash
iptables -t nat -A PREROUTING \
    -d 138.4.83.140 \
    -p tcp --dport 443 \
    -j REDIRECT --to-ports 8006
```

### 3. Redirección para conexiones originadas en el propio servidor

```bash
iptables -t nat -A OUTPUT \
    -d 138.4.83.140 \
    -p tcp --dport 443 \
    -j REDIRECT --to-ports 8006
```

### 4. Comprobar las reglas

```bash
iptables -t nat -L -n -v
```

Deberías obtener algo parecido a:

```text
Chain PREROUTING
REDIRECT  tcp  --  0.0.0.0/0  138.4.83.140  tcp dpt:443 redir ports 8006

Chain OUTPUT
REDIRECT  tcp  --  0.0.0.0/0  138.4.83.140  tcp dpt:443 redir ports 8006
```

### 5. Verificar el funcionamiento

Desde el propio servidor:

```bash
curl -vk https://138.4.83.140/
```

Y desde otro equipo:

```
https://138.4.83.140/
```

---

## Para hacerlas permanentes en Proxmox

Si utilizas `/etc/network/interfaces`, añade únicamente estas dos líneas en la configuración de la interfaz (`vmbr0` normalmente):

```text
post-up iptables -t nat -A PREROUTING -d 138.4.83.140 -p tcp --dport 443 -j REDIRECT --to-ports 8006
post-up iptables -t nat -A OUTPUT -d 138.4.83.140 -p tcp --dport 443 -j REDIRECT --to-ports 8006
```

Y las correspondientes para eliminarlas al bajar la interfaz:

```text
post-down iptables -t nat -D PREROUTING -d 138.4.83.140 -p tcp --dport 443 -j REDIRECT --to-ports 8006
post-down iptables -t nat -D OUTPUT -d 138.4.83.140 -p tcp --dport 443 -j REDIRECT --to-ports 8006
```

Esta es la configuración más sencilla y adecuada para publicar la interfaz web de Proxmox por el puerto **443** sin utilizar reglas de marcado (`MARK`).




Si tienes **Proxmox VE 9 (Debian 13 Trixie)** y **no dispones de suscripción Enterprise**, esta es la receta completa para cambiar correctamente los repositorios.

## 1. Deshabilitar los repositorios Enterprise

```bash
mkdir -p /root/apt-backup

[ -f /etc/apt/sources.list.d/pve-enterprise.sources ] && \
mv /etc/apt/sources.list.d/pve-enterprise.sources /root/apt-backup/

[ -f /etc/apt/sources.list.d/ceph.sources ] && \
mv /etc/apt/sources.list.d/ceph.sources /root/apt-backup/
```

## 2. Crear el repositorio oficial No-Subscription

```bash
cat >/etc/apt/sources.list.d/pve-no-subscription.sources <<'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
```

## 3. (Opcional) Si utilizas Ceph

**Solo si tienes un clúster Ceph instalado:**

```bash
cat >/etc/apt/sources.list.d/ceph.sources <<'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/ceph-squid
Suites: trixie
Components: no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
```

Si no utilizas Ceph, **no crees este archivo**.

## 4. Actualizar la información de los repositorios

```bash
apt update
```

## 5. Actualizar el sistema

```bash
apt full-upgrade -y
```

## 6. Reiniciar (si se actualiza el kernel)

```bash
reboot
```

---

## Comprobación final

Verifica que solo quedan activos los repositorios correctos:

```bash
find /etc/apt/sources.list.d -maxdepth 1 -type f -print -exec cat {} \;
```

Deberías ver únicamente:

* `debian.sources`
* `pve-no-subscription.sources`
* `ceph.sources` **solo si realmente utilizas Ceph**

Y al ejecutar:

```bash
apt update
```

no deben aparecer errores `401 Unauthorized` ni avisos sobre repositorios Enterprise.


Aquí tienes una **receta completa** para configurar un almacenamiento **ZFS RAIDZ1** en Proxmox con **1 SSD de 500 GB para el sistema** y **3 HDD de 4 TB para datos**.

```bash
###############################################################################
# 1. Identificar los discos
###############################################################################

lsblk -o NAME,SIZE,MODEL

ls -l /dev/disk/by-id/

###############################################################################
# 2. Borrar firmas anteriores (¡¡¡ELIMINA TODOS LOS DATOS!!)
###############################################################################

wipefs -a /dev/sdb
wipefs -a /dev/sdc
wipefs -a /dev/sdd

sgdisk --zap-all /dev/sdb
sgdisk --zap-all /dev/sdc
sgdisk --zap-all /dev/sdd

###############################################################################
# 3. Crear el pool ZFS RAIDZ1
###############################################################################

zpool create -f \
    -o ashift=12 \
    datos \
    raidz1 \
    /dev/disk/by-id/ata-ST4000NT001-3M2101_WX1232Q5 \
    /dev/disk/by-id/ata-ST4000NT001-3M2101_WX1232P2 \
    /dev/disk/by-id/ata-ST4000NT001-3M2101_WX1232V0

###############################################################################
# 4. Comprobar el estado del pool
###############################################################################

zpool status

zpool list

zfs list

###############################################################################
# 5. Crear datasets
###############################################################################

zfs create datos/vmdata
zfs create datos/backup
zfs create datos/iso
zfs create datos/templates

###############################################################################
# 6. Optimizar el dataset para máquinas virtuales
###############################################################################

zfs set compression=lz4 datos/vmdata
zfs set atime=off datos/vmdata
zfs set xattr=sa datos/vmdata
zfs set acltype=posixacl datos/vmdata
zfs set dnodesize=auto datos/vmdata
zfs set recordsize=16K datos/vmdata

###############################################################################
# 7. Optimizar datasets de backup e ISOs
###############################################################################

zfs set compression=lz4 datos/backup
zfs set compression=lz4 datos/iso
zfs set compression=lz4 datos/templates

###############################################################################
# 8. Añadir el almacenamiento a Proxmox
###############################################################################

pvesm add zfspool vmdata \
    --pool datos/vmdata \
    --content images,rootdir

###############################################################################
# 9. Verificar
###############################################################################

pvesm status

zfs list

zpool status
```

## Resultado esperado

```
datos
├── vmdata
│   ├── Máquinas virtuales
│   └── Contenedores LXC
├── backup
│   └── Copias de seguridad
├── iso
│   └── Imágenes ISO
└── templates
    └── Plantillas de contenedores
```

## Capacidad

Con tres discos de **4 TB** en **RAIDZ1** obtendrás aproximadamente:

* **Capacidad útil:** ≈ **7,2 TiB** (unos **8 TB** comerciales).
* **Tolerancia a fallos:** **1 disco**.

# Descargar una imagen ISO en Proxmox VE

Este documento describe las diferentes formas de descargar e instalar imágenes ISO en un servidor **Proxmox VE**, para posteriormente utilizarlas en la creación de máquinas virtuales.

---

# Requisitos

- Proxmox VE instalado y operativo.
- Acceso a la interfaz web o a la consola del servidor.
- Conexión a Internet.

---

# Método 1. Descargar una ISO desde la interfaz web

1. Acceder a la interfaz web de Proxmox.

2. Seleccionar el nodo.

   ```
   Datacenter
     └── <Nodo Proxmox>
   ```

3. Seleccionar el almacenamiento **local**.

4. Abrir la pestaña:

   ```
   ISO Images
   ```

5. Pulsar:

   ```
   Download from URL
   ```

6. Introducir la URL oficial de la ISO.

   Ejemplo Ubuntu Server 24.04 LTS:

   ```
   https://releases.ubuntu.com/24.04/ubuntu-24.04.3-live-server-amd64.iso
   ```

7. Pulsar **Query URL**.

8. Finalmente pulsar **Download**.

La descarga comenzará automáticamente.

---

# Método 2. Subir una ISO desde el ordenador

1. Seleccionar:

   ```
   Datacenter
     └── Nodo
         └── local
             └── ISO Images
   ```

2. Pulsar:

   ```
   Upload
   ```

3. Seleccionar el fichero ISO.

4. Esperar a que finalice la carga.

---

# Método 3. Descargar una ISO desde la consola

Acceder por SSH al servidor.

Entrar en el directorio de almacenamiento de ISOs.

```bash
cd /var/lib/vz/template/iso
```

Descargar la ISO mediante `wget`.

Ejemplo:

```bash
wget https://releases.ubuntu.com/24.04/ubuntu-24.04.3-live-server-amd64.iso
```

También puede utilizarse `curl`:

```bash
curl -LO https://releases.ubuntu.com/24.04/ubuntu-24.04.3-live-server-amd64.iso
```

---

# Método 4. Copiar una ISO mediante SCP

Desde otro equipo Linux:

```bash
scp ubuntu-24.04.3-live-server-amd64.iso \
root@<IP_PROXMOX>:/var/lib/vz/template/iso/
```

Ejemplo:

```bash
scp ubuntu-24.04.3-live-server-amd64.iso \
root@138.4.83.140:/var/lib/vz/template/iso/
```

---

# Método 5. Copiar mediante WinSCP

1. Abrir WinSCP.

2. Conectarse al servidor Proxmox mediante SFTP.

3. Navegar hasta:

```
/var/lib/vz/template/iso/
```

4. Arrastrar la ISO al directorio.

---

# Comprobar las ISOs instaladas

Desde la consola:

```bash
ls -lh /var/lib/vz/template/iso/
```

Ejemplo:

```text
-rw-r--r-- 1 root root 2.9G ubuntu-24.04.3-live-server-amd64.iso
-rw-r--r-- 1 root root 1.2G debian-13.0.0-amd64-netinst.iso
```

---

# Utilizar una ISO al crear una máquina virtual

1. Pulsar **Create VM**.

2. En la pestaña **OS** seleccionar:

```
Use CD/DVD disc image file (iso)
```

3. Elegir la ISO descargada.

4. Continuar con el asistente de creación.

---

# Ubicación de las imágenes ISO

Por defecto, Proxmox almacena las ISOs en:

```text
/var/lib/vz/template/iso/
```

---

# Repositorios oficiales recomendados

## Ubuntu

https://ubuntu.com/download/server

## Debian

https://www.debian.org/download

## Rocky Linux

https://rockylinux.org/download

## AlmaLinux

https://almalinux.org

## Fedora Server

https://fedoraproject.org/server

## openSUSE

https://get.opensuse.org

## Proxmox VE

https://www.proxmox.com/en/downloads

---

# Verificación de integridad

Se recomienda verificar la suma SHA256 de la imagen descargada.

Ejemplo:

```bash
sha256sum ubuntu-24.04.3-live-server-amd64.iso
```

Comparar el resultado con el publicado por el fabricante antes de utilizar la imagen.

# Crear una máquina virtual Ubuntu Server 24.04 LTS en Proxmox VE

## Objetivo

Crear una máquina virtual optimizada para alojar un servidor WordPress sobre Ubuntu Server 24.04 LTS utilizando almacenamiento ZFS en Proxmox VE.

---

# Requisitos

- Proxmox VE 9.x
- Imagen ISO Ubuntu Server 24.04 LTS descargada
- Almacenamiento ZFS configurado
- Acceso como administrador a Proxmox

---

# Paso 1. Crear la máquina virtual

Desde la interfaz web:

```
Datacenter
    └── virtualizacion1
            └── Create VM
```

---

# Ventana 1. General

## Configuración básica

| Opción | Valor |
|---------|-------|
| Node | virtualizacion1 |
| VM ID | Automático o siguiente disponible |
| Name | wordpress-01 |
| Resource Pool | (Vacío) |
| Add to HA | No |

## Opciones avanzadas

Activar **Advanced**.

| Opción | Valor |
|---------|-------|
| Start at boot | Sí |
| Start/Shutdown order | 1 |
| Startup delay | 30 |
| Shutdown timeout | 180 |
| vCPU Architecture | Default (Host Architecture) |
| Tags | wordpress, ubuntu (opcional) |

Pulsar **Next**.

---

# Ventana 2. OS

Seleccionar el medio de instalación.

| Opción | Valor |
|---------|-------|
| Use CD/DVD disc image file (iso) | Sí |
| Storage | local |
| ISO Image | ubuntu-24.04.x-live-server-amd64.iso |
| Guest OS | Linux |
| Version | 7.x - 2.6 Kernel |

Pulsar **Next**.

---

# Ventana 3. System

Configurar el hardware virtual.

| Opción | Valor |
|---------|-------|
| Graphic Card | Default |
| Machine | q35 |
| BIOS | OVMF (UEFI) |
| Add EFI Disk | Sí |
| EFI Storage | local-lvm |
| Pre-Enroll Keys | Sí |
| SCSI Controller | VirtIO SCSI single |
| QEMU Agent | Sí |
| Add TPM | No |

Pulsar **Next**.

---

# Ventana 4. Disks

Crear el disco virtual.

## Configuración principal

| Opción | Valor |
|---------|-------|
| Bus/Device | SCSI |
| SCSI Controller | VirtIO SCSI single |
| Storage | datos |
| Disk size | 80 GiB |
| Format | Raw disk image (raw) |
| Cache | Default (No cache) |
| Discard | Sí |
| IO Thread | Sí |

## Opciones avanzadas

| Opción | Valor |
|---------|-------|
| SSD Emulation | Sí |
| Backup | Sí |
| Read-only | No |
| Skip replication | No |
| Async IO | Default (io_uring) |

Pulsar **Next**.

---

# Ventana 5. CPU

## Configuración principal

| Opción | Valor |
|---------|-------|
| Sockets | 1 |
| Cores | 4 |
| Type | host |

## Opciones avanzadas

| Opción | Valor |
|---------|-------|
| VCPUs | 4 |
| CPU Units | 100 |
| CPU Limit | unlimited |
| CPU Affinity | All Cores |
| Enable NUMA | No |
| Extra CPU Flags | Default |

Pulsar **Next**.

---

# Ventana 6. Memory

| Opción | Valor |
|---------|-------|
| Memory | 4096 MiB |
| Minimum Memory | 2048 MiB |
| Ballooning Device | Sí |
| Allow KSM | Sí |

Pulsar **Next**.

---

# Ventana 7. Network

| Opción | Valor |
|---------|-------|
| No network device | No |
| Bridge | vmbr0 |
| Model | VirtIO (paravirtualized) |
| MAC Address | Auto |
| VLAN Tag | Sin VLAN |
| Firewall | No |
| Disconnect | No |
| MTU | Same as bridge |
| Rate limit | Unlimited |
| Multiqueue | Default |

Pulsar **Next**.

---

# Ventana 8. Confirm

Revisar toda la configuración.

Activar:

```
Start after created
```

Pulsar:

```
Finish
```

---

# Configuración final

| Recurso | Valor |
|----------|-------|
| Sistema operativo | Ubuntu Server 24.04 LTS |
| Firmware | UEFI (OVMF) |
| Chipset | q35 |
| Disco | 80 GiB RAW |
| Almacenamiento | datos (ZFS) |
| CPU | 4 vCPU (host) |
| RAM | 4 GB |
| Red | VirtIO |
| QEMU Guest Agent | Activado |

---

# Justificación de la configuración

Esta configuración proporciona:

- Hardware virtual moderno (UEFI + q35).
- Máximo rendimiento en CPU mediante el tipo **host**.
- Excelente rendimiento de disco sobre ZFS mediante **RAW**, **VirtIO SCSI**, **Discard**, **IO Thread** e **io_uring**.
- Compatibilidad con TRIM y snapshots.
- Arranque automático tras reiniciar el servidor Proxmox.
- Integración completa con Proxmox mediante **QEMU Guest Agent**.
- Configuración preparada para formar parte de un futuro clúster Proxmox con un segundo nodo de hardware idéntico.

---

# Siguiente paso

Una vez creada la máquina virtual:

1. Instalar Ubuntu Server 24.04 LTS.
2. Actualizar el sistema.
3. Instalar `qemu-guest-agent`.
4. Configurar el acceso SSH.
5. Instalar la pila web (LEMP o LAMP).
6. Instalar WordPress.
7. Configurar HTTPS mediante Let's Encrypt.
8. Crear un snapshot inicial de la máquina virtual.
