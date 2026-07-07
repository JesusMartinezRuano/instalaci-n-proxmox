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

## Recomendación para tu servidor

Dado que ya tienes:

* **SSD WD 500 GB** → Proxmox VE.
* **3 × Seagate IronWolf Pro 4 TB** → Pool `datos`.

Te recomiendo utilizar:

* **`local-lvm` (SSD)** para las VMs que necesiten mucho rendimiento (por ejemplo, Windows o bases de datos muy exigentes).
* **`vmdata` (ZFS RAIDZ1)** para:

  * Máquinas virtuales Linux.
  * Contenedores LXC.
  * Docker.
  * Moodle.
  * Open WebUI.
  * Ollama.
  * Almacenamiento general.

Esta distribución aprovecha el rendimiento del SSD para el sistema y reserva el RAIDZ1 para ofrecer una gran capacidad con protección frente al fallo de un disco.

