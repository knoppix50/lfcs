---
title: 02. Configuración de Redes
nav_order: 3
---

# 02. Configuración de Redes
{: .no_toc }

Configuración persistente con `nmcli` y `netplan`, diagnóstico de interfaces y rutas con `ip`, análisis de sockets con `ss`, resolución DNS, sincronización horaria y túneles SSH.
{: .fs-6 .fw-300 }

## Contenido
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 1. Configuración Persistente de Red

En el examen LFCS es obligatorio que cualquier cambio de red sobreviva a un reinicio. Debes dominar **NetworkManager (`nmcli`)** en sistemas RHEL/Rocky y **Netplan** en sistemas Ubuntu.

### NetworkManager (`nmcli` - RHEL / Rocky Linux)

| Acción | Comando |
| :--- | :--- |
| **Ver dispositivos físicos** | `nmcli device status` |
| **Listar perfiles de conexión** | `nmcli connection show` |
| **Crear conexión IP estática** | `nmcli con add con-name static-eth0 ifname eth0 type ethernet ipv4.addresses 192.168.1.50/24 ipv4.gateway 192.168.1.1 ipv4.dns "8.8.8.8 1.1.1.1" ipv4.method manual` |
| **Añadir segunda IP (Alias)** | `nmcli con mod static-eth0 +ipv4.addresses 192.168.1.51/24` |
| **Añadir ruta estática** | `nmcli con mod static-eth0 +ipv4.routes "10.10.0.0/16 192.168.1.254"` |
| **Activar / Aplicar cambios** | `nmcli con up static-eth0` |
| **Eliminar perfil** | `nmcli con delete static-eth0` |

{: .note }
> **Truco de examen:** Si modificas una conexión activa con `nmcli con mod`, los cambios **no se aplican** en caliente hasta que ejecutas `nmcli con up <nombre_conexion>`.

### Netplan (Ubuntu Server)

Los archivos YAML se ubican en `/etc/netplan/` (ej. `01-netcfg.yaml`) y requieren permisos restrictivos (`chmod 600`).

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      addresses:
        - 192.168.1.50/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
```

```bash
# Validar sintaxis antes de aplicar (revierte automáticamente si pierdes conexión)
netplan try

# Aplicar configuración definitiva
netplan apply
```

---

## 2. Diagnóstico de Interfaces y Enrutamiento (`ip`)

La suite `iproute2` sustituye a los comandos obsoletos (`ifconfig`, `route`, `arp`).

```bash
# Ver direcciones IP (formato compacto y con colores)
ip -br -c a

# Ver estado de enlaces (UP/DOWN) y direcciones MAC
ip -br -c link

# Levantar o tirar una interfaz en caliente
ip link set dev eth0 up
ip link set dev eth0 down

# Añadir IP temporal (se pierde al reiniciar)
ip addr add 10.0.0.15/24 dev eth0

# Ver tabla de enrutamiento y tabla de vecinos (ARP)
ip route show
ip neigh show

# Consultar qué interfaz y gateway usará el kernel para llegar a una IP destino
ip route get 8.8.8.8
```

---

## 3. Análisis de Puertos, Sockets y Tráfico (`ss`, `nc`, `tcpdump`)

### Inspección de sockets con `ss`

```bash
# Ver todos los puertos TCP y UDP en escucha (LISTEN) con su proceso y PID
ss -tulnp

# Ver conexiones TCP establecidas filtrando por puerto destino
ss -tan state established '( dport = :443 or sport = :443 )'

# Resumen estadístico de sockets
ss -s
```

### Pruebas de conectividad y captura de paquetes

```bash
# Comprobar si un puerto TCP remoto está abierto y accesible (timeout 3s)
nc -zv -w 3 192.168.1.100 22

# Levantar un servidor TCP de prueba en el puerto 8080 para verificar firewall/red
nc -l -p 8080

# Capturar tráfico ICMP o de un puerto específico en una interfaz
tcpdump -i eth0 -nn port 80
tcpdump -i any -nn icmp
```

---

## 4. Resolución de Nombres (DNS) y Hostname

### Configuración de Hostname y `/etc/hosts`

```bash
# Cambiar el FQDN del servidor de forma persistente
hostnamectl set-hostname web01.empresa.local

# Verificar resolución local vs DNS
getent hosts web01.empresa.local
```

{: .warning }
> **Orden de resolución (`/etc/nsswitch.conf`):** La línea `hosts: files dns` indica que el sistema lee primero `/etc/hosts` antes de consultar los servidores DNS configurados en `/etc/resolv.conf`.

### Consultas DNS con `dig`

```bash
# Consulta directa obteniendo solo la IP (ideal para scripts)
dig +short example.com A

# Consultar registros MX (correo) o NS (servidores de nombres)
dig example.com MX +noall +answer

# Consultar a un servidor DNS específico (ej. 1.1.1.1)
dig @1.1.1.1 example.com

# Resolución inversa (IP a nombre)
dig -x 8.8.8.8 +short
```

---

## 5. Sincronización Horaria (`timedatectl` y `chrony`)

Una desincronización horaria rompe certificados TLS, Kerberos y logs distribuidos.

```bash
# Ver estado de reloj, zona horaria y sincronización NTP
timedatectl status

# Configurar zona horaria
timedatectl list-timezones | grep Madrid
timedatectl set-timezone Europe/Madrid

# Habilitar sincronización NTP
timedatectl set-ntp true

# Verificar fuentes NTP activas y estado de sincronización en chrony
chronyc sources -v
chronyc tracking
```

---

## 6. SSH Seguro y Túneles (Port Forwarding)

### Endurecimiento del servidor (`/etc/ssh/sshd_config`)

```ini
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers adminuser deploy
```

```bash
# Validar sintaxis de sshd_config ANTES de reiniciar el servicio (evita quedarte fuera)
sshd -t
systemctl reload sshd
```

### Gestión de llaves y Túneles SSH

```bash
# Generar par de llaves Ed25519
ssh-keygen -t ed25519 -C "admin@lfcs"

# Copiar llave pública al servidor remoto
ssh-copy-id -i ~/.ssh/id_ed25519.pub usuario@192.168.1.50

# Túnel Local (-L): Acceder a una BD remota (3306) como si estuviera en mi localhost:8806
ssh -f -N -L 8806:127.0.0.1:3306 usuario@servidor_remoto

# Túnel Remoto (-R): Exponer un servicio de mi máquina local (80) en el puerto 9090 del servidor remoto
ssh -f -N -R 9090:127.0.0.1:80 usuario@servidor_remoto
```

---

## 7. Interfaces Virtuales: Bonding y VLANs (`nmcli`)

En escenarios avanzados de red pueden pedirte agregar redundancia de enlaces (*bonding* o *teaming*) o etiquetado VLAN 802.1Q.

```bash
# Crear interfaz Bond (modo active-backup)
nmcli con add type bond con-name bond0 ifname bond0 bond.options "mode=active-backup,miimon=100"
nmcli con add type ethernet slave-type bond con-name bond0-p1 ifname eth1 master bond0
nmcli con add type ethernet slave-type bond con-name bond0-p2 ifname eth2 master bond0
nmcli con mod bond0 ipv4.addresses 10.20.0.10/24 ipv4.method manual
nmcli con up bond0

# Crear subinterfaz VLAN 100 sobre eth0
nmcli con add type vlan con-name vlan100 ifname eth0.100 dev eth0 id 100 ipv4.addresses 172.16.100.5/24 ipv4.method manual
nmcli con up vlan100
```

