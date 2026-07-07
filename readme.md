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



Aquí tienes las reglas eliminando el prefijo `post-up`, listas para ejecutarlas directamente en la consola:

```bash
iptables -t nat -A PREROUTING -p tcp -d 138.4.83.140 --dport 443 -j REDIRECT --to-ports 8006

iptables -t nat -A OUTPUT -p tcp -d 138.4.83.140 --dport 443 -j REDIRECT --to-ports 8006
```

Si las vas a utilizar en un servidor Proxmox, recuerda sustituir:

```text
<your-ip>
```

por la dirección IP pública o la IP del puente (`vmbr0`) donde recibe las conexiones HTTPS, por ejemplo:

```bash
iptables -t nat -A PREROUTING -d 192.168.1.10 -p tcp --dport 443 -j REDIRECT --to-port 8006
```

O:

```bash
iptables -t nat -A PREROUTING -d 203.0.113.25 -p tcp --dport 443 -j REDIRECT --to-port 8006
```

Si lo que pretendes es publicar la interfaz web de Proxmox por el puerto 443, existe una forma más sencilla y robusta utilizando únicamente `DNAT` o configurando un proxy inverso como Nginx o Caddy, evitando el uso de marcas (`MARK`) y reglas adicionales de filtrado.



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
