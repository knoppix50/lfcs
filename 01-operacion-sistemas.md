---
title: 01. Operación de Sistemas
nav_order: 2
---

# 01. Operación de Sistemas
{: .no_toc }

Gestión de servicios con `systemd`, control de procesos, análisis de logs, tareas programadas, parámetros del kernel y resolución de problemas de arranque.
{: .fs-6 .fw-300 }

## Contenido
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 1. Gestión de Servicios y Unidades (`systemd`)

En el examen LFCS es crítico dominar el ciclo de vida de los servicios, la creación de *drop-in files* para sobrescribir configuraciones sin tocar los archivos originales del paquete y la gestión de *targets*.

### Comandos esenciales de `systemctl`

| Acción | Comando | Descripción |
| :--- | :--- | :--- |
| **Estado / Logs recientes** | `systemctl status nginx -l --no-pager` | Muestra estado completo sin truncar líneas. |
| **Iniciar y habilitar** | `systemctl enable --now httpd` | Arranca el servicio y crea el symlink para el boot. |
| **Enmascarar (Bloquear)** | `systemctl mask firewalld` | Apunta el servicio a `/dev/null` (impide inicio manual o por dependencia). |
| **Desenmascarar** | `systemctl unmask firewalld` | Revierte el bloqueo de `mask`. |
| **Recargar daemon** | `systemctl daemon-reload` | Obligatorio tras crear o modificar archivos `.service`. |
| **Listar fallidos** | `systemctl --failed` | Muestra unidades en estado `failed`. |
| **Ver dependencias** | `systemctl list-dependencies sshd` | Árbol de dependencias (`Requires`, `Wants`). |

### Modificación de servicios (*Drop-in files*) y creación de unidades

Nunca edites `/usr/lib/systemd/system/servicio.service` directamente; las actualizaciones del sistema sobrescribirán los cambios.

```bash
# Crear override limpio (genera /etc/systemd/system/nginx.service.d/override.conf)
systemctl edit nginx

# Ejemplo de contenido dentro del override para reinicio automático:
[Service]
Restart=always
RestartSec=5s

# Ver la configuración efectiva combinada (archivo base + overrides)
systemctl cat nginx
```

{: .note }
> **Ubicación de prioridades en systemd:** `/etc/systemd/system/` (máxima prioridad, admin) > `/run/systemd/system/` (runtime) > `/usr/lib/systemd/system/` (paquetes del sistema).

### Gestión de Targets (Runlevels)

```bash
# Ver el target por defecto actual
systemctl get-default

# Cambiar a modo consola multiusuario (sin entorno gráfico)
systemctl set-default multi-user.target

# Cambiar a modo rescate (monta sistemas de archivos, sin red ni servicios)
systemctl isolate rescue.target
```

---

## 2. Control de Procesos y Prioridades

La monitorización y ajuste de recursos de procesos en ejecución es una tarea recurrente.

### Inspección y señales (`ps`, `pgrep`, `kill`)

```bash
# Top 5 procesos que más CPU consumen
ps -eo pid,ppid,user,%cpu,%mem,cmd --sort=-%cpu | head -n 6

# Top 5 procesos que más RAM consumen
ps -eo pid,ppid,user,%cpu,%mem,cmd --sort=-%mem | head -n 6

# Buscar PID exacto por nombre o patrón completo
pgrep -af apache2

# Enviar señales (SIGTERM=15 elegante, SIGKILL=9 forzado, SIGHUP=1 recargar config)
kill -15 <PID>
pkill -HUP -u usuario_app proceso
```

### Prioridad de planificación (`nice` y `renice`)

El rango de *niceness* va de **`-20`** (máxima prioridad) a **`19`** (mínima prioridad). Por defecto, los procesos nacen con `0`.

```bash
# Lanzar un proceso de backup con baja prioridad (nice 15)
nice -n 15 tar -czf /backup/data.tar.gz /var/www

# Modificar la prioridad de un proceso ya en ejecución (solo root puede poner valores negativos)
renice -n -5 -p <PID>

# Cambiar prioridad a todos los procesos de un usuario
renice -n 10 -u devuser
```

---

## 3. Análisis de Logs (`journalctl` y `rsyslog`)

### Consultas avanzadas con `journalctl`

```bash
# Filtrar por unidad específica y mostrar solo errores o superior (emerg, alert, crit, err)
journalctl -u sshd -p err

# Logs desde el último arranque (-b) o el anterior (-b -1)
journalctl -b -1 -p warning

# Filtrar por rango de tiempo exacto
journalctl --since "2026-11-12 10:00:00" --until "2026-11-12 12:30:00"

# Seguimiento en tiempo real (equivalente a tail -f)
journalctl -u nginx -f
```

### Persistencia del Journal

En algunas instalaciones mínimas, `systemd-journald` guarda los logs solo en RAM (`/run/log/journal`). Para hacerlos persistentes tras reiniciar:

```bash
mkdir -p /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal
systemctl restart systemd-journald
```

{: .warning }
> **Diferencia de rutas de logs tradicionales (`rsyslog`):**
> * **RHEL / Rocky Linux:** Autenticación en `/var/log/secure` y mensajes generales en `/var/log/messages`.
> * **Debian / Ubuntu:** Autenticación en `/var/log/auth.log` y mensajes generales en `/var/log/syslog`.

---

## 4. Tareas Programadas (`cron`, `at` y `systemd timers`)

### Crontab de usuario y sistema

Formato de `crontab -e`: `MIN HORA DIA_MES MES DIA_SEMANA COMANDO`

```bash
# Ejecutar script cada 15 minutos de lunes a viernes
*/15 * * * 1-5 /usr/local/bin/check_disk.sh >> /var/log/disk_check.log 2>&1

# Ejecutar al arrancar el sistema
@reboot /usr/local/bin/init_app.sh

# Editar crontab de otro usuario (como root)
crontab -u nginx -e
```

* **Control de acceso a cron:** Si existe `/etc/cron.allow`, **solo** los usuarios listados ahí pueden usar `cron`. Si no existe, se consulta `/etc/cron.deny`.
* **Tareas únicas (`at`):**
  ```bash
  echo "/usr/local/bin/maintenance.sh" | at 02:30 tomorrow
  atq          # Listar cola de tareas pendientes
  atrm <ID>    # Eliminar tarea pendiente
  ```

### Systemd Timers (Alternativa moderna a cron)

Un timer requiere dos archivos con el mismo nombre base en `/etc/systemd/system/`: `backup.service` (qué ejecutar) y `backup.timer` (cuándo ejecutarlo).

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Ejecutar backup diario a las 03:00

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
systemctl daemon-reload
systemctl enable --now backup.timer
systemctl list-timers --all
```

---

## 5. Parámetros del Kernel y Módulos

### Ajuste en caliente y persistente (`sysctl`)

```bash
# Ver todos los parámetros actuales
sysctl -a | grep ip_forward

# Cambiar valor en tiempo real (se pierde al reiniciar)
sysctl -w net.ipv4.ip_forward=1

# Hacerlo persistente (crear archivo en /etc/sysctl.d/)
echo "net.ipv4.ip_forward = 1" > /etc/sysctl.d/99-routing.conf
sysctl --system   # Aplica todos los archivos de /etc/sysctl.d/
```

### Gestión de módulos del kernel (`lsmod`, `modprobe`)

```bash
# Listar módulos cargados
lsmod

# Cargar un módulo y sus dependencias
modprobe br_netfilter

# Descargar un módulo
modprobe -r br_netfilter

# Cargar módulo automáticamente en cada arranque
echo "br_netfilter" > /etc/modules-load.d/k8s.conf

# Bloquear (blacklist) la carga de un módulo (ej. usb-storage o nouveau)
echo "blacklist usb-storage" > /etc/modprobe.d/disable-usb.conf
```

---

## 6. Arrancador (GRUB2) y Recuperación de Sistema

En el examen es habitual encontrar máquinas que no arrancan por errores en `/etc/fstab` o contraseñas de `root` perdidas.

### Actualizar configuración de GRUB2

Tras modificar `/etc/default/grub` (por ejemplo, cambiar `GRUB_TIMEOUT` o parámetros en `GRUB_CMDLINE_LINUX`):

| Distribución | Comando para regenerar configuración de GRUB2 |
| :--- | :--- |
| **Debian / Ubuntu** | `update-grub` (o `grub-mkconfig -o /boot/grub/grub.cfg`) |
| **RHEL / Rocky (BIOS)** | `grub2-mkconfig -o /boot/grub2/grub.cfg` |
| **RHEL / Rocky (UEFI)** | `grub2-mkconfig -o /boot/efi/EFI/rocky/grub.cfg` *(o `grub2-mkconfig -o /boot/grub2/grub.cfg` en RHEL 9+)* |

### Recuperar contraseña de `root` desde GRUB2

1. Reiniciar la máquina y pulsar **`e`** en el menú de GRUB para editar la entrada del kernel.
2. Buscar la línea que empieza por `linux` (o `linux16`/`linuxefi`).
3. Añadir al final de esa línea **`rd.break`** (en RHEL/Rocky) o **`init=/bin/bash`** (en Debian/Ubuntu).
4. Pulsar **`Ctrl + X`** para arrancar.
5. Ejecutar la secuencia de montaje y cambio de contraseña:

```bash
# --- Si usaste rd.break (RHEL / Rocky Linux) ---
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel    # CRÍTICO: fuerza reetiquetado SELinux al reiniciar
exit
reboot

# --- Si usaste init=/bin/bash (Debian / Ubuntu) ---
mount -o remount,rw /
passwd root
sync
exec /sbin/init
```

{: .warning }
> **¡Cuidado con SELinux!** Si olvidas `touch /.autorelabel` en RHEL/Rocky tras cambiar `/etc/shadow` desde `rd.break`, el archivo perderá su contexto SELinux (`shadow_t`) y el sistema rechazará cualquier inicio de sesión después del reinicio.

