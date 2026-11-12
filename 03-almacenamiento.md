---
title: 03. Almacenamiento y Sistemas de Archivos
nav_order: 4
---

# 03. Almacenamiento y Sistemas de Archivos
{: .no_toc }

Particionado de discos (MBR/GPT), gestión completa de volúmenes lógicos (LVM), creación y redimensionado de sistemas de archivos (`ext4`/`xfs`), memoria Swap, montaje persistente (`/etc/fstab`) y cuotas de disco.
{: .fs-6 .fw-300 }

## Contenido
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 1. Inspección de Dispositivos de Bloque

Antes de tocar cualquier disco o partición, identifica siempre la topología exacta, los UUIDs y los puntos de montaje actuales.

```bash
# Vista en árbol de discos, particiones, LVM y puntos de montaje con tipos de FS y UUID
lsblk -f

# Ver tabla de particiones exacta (MBR vs GPT) y tamaños de sector
fdisk -l

# Obtener exclusivamente el UUID y TYPE de cada dispositivo (clave para /etc/fstab)
blkid

# Ver uso de espacio en disco (humano y mostrando tipo de sistema de archivos)
df -hT

# Ver uso de inodos (vital cuando hay espacio libre pero falla la creación de archivos)
df -i

# Calcular tamaño de directorios (profundidad 1, ordenado)
du -h --max-depth=1 /var | sort -hr
```

---

## 2. Particionado de Discos (`parted` y `gdisk`/`fdisk`)

En el examen LFCS puedes encontrar discos mayores de 2 TB (requieren **GPT**) o sistemas heredados (**MBR/dos**).

### Particionado no interactivo con `parted` (Ideal para rapidez en examen)

```bash
# Inicializar tabla de particiones GPT en /dev/sdb
parted -s /dev/sdb mklabel gpt

# Crear partición primaria de 2 GB
parted -s /dev/sdb mkpart primary ext4 1MiB 2049MiB

# Crear partición con el 100% del espacio restante y marcarla con flag LVM
parted -s /dev/sdb mkpart primary 2049MiB 100%
parted -s /dev/sdb set 2 lvm on

# Notificar al kernel los cambios en la tabla de particiones sin reiniciar
partprobe /dev/sdb
```

### Códigos de tipo clave en `fdisk` / `gdisk`

| Tipo de partición | Código MBR (`fdisk`) | Código GPT (`gdisk`) |
| :--- | :--- | :--- |
| **Linux Filesystem** | `83` | `8300` |
| **Linux Swap** | `82` | `8200` |
| **Linux LVM** | `8e` | `8e00` |
| **Linux RAID** | `fd` | `fd00` |

---

## 3. Gestión de Volúmenes Lógicos (LVM)

El ciclo de vida de LVM sigue siempre la jerarquía: **Dispositivo físico -> Physical Volume (PV) -> Volume Group (VG) -> Logical Volume (LV) -> Filesystem**.

### Creación desde cero

```bash
# 1. Inicializar discos o particiones como Physical Volumes
pvcreate /dev/sdb1 /dev/sdc1

# 2. Crear un Volume Group llamado 'vg_data' (opcional: tamaño de Physical Extent -s 16M)
vgcreate vg_data /dev/sdb1 /dev/sdc1

# 3. Crear Logical Volumes (por tamaño fijo o porcentaje de espacio libre)
lvcreate -L 5G -n lv_app vg_data
lvcreate -l 100%FREE -n lv_logs vg_data

# Inspección rápida en una línea de cada capa
pvs && vgs && lvs
```

### Redimensionado de Volúmenes Lógicos (Operación crítica en examen)

El parámetro **`-r` (`--resizefs`)** es tu mejor aliado: redimensiona simultáneamente el volumen lógico y el sistema de archivos subyacente sin errores de cálculo.

```bash
# Añadir un nuevo disco al Volume Group existente
pvcreate /dev/sdd
vgextend vg_data /dev/sdd

# Extender el LV en +2 GB y redimensionar el sistema de archivos automáticamente
lvextend -r -L +2G /dev/vg_data/lv_app

# Extender usando todo el espacio libre restante del VG
lvextend -r -l +100%FREE /dev/vg_data/lv_app
```

{: .warning }
> **¡Cuidado al reducir volúmenes (`lvreduce`)!**
> * **XFS NO se puede reducir.** Si te piden reducir un volumen XFS, debes hacer backup de datos, destruir/recrear el LV más pequeño, formatear de nuevo y restaurar.
> * **EXT4 SÍ se puede reducir**, pero **nunca en caliente** (requiere desmontar antes con `umount`): `lvreduce -r -L 3G /dev/vg_data/lv_app`.

---

## 4. Sistemas de Archivos (`ext4` vs `xfs`)

### Formateo y etiquetado

```bash
# Formatear en ext4 asignando una etiqueta (LABEL)
mkfs.ext4 -L DATA_APP /dev/vg_data/lv_app

# Formatear en XFS (forzando sobrescritura si ya existía firma previa con -f)
mkfs.xfs -f -L LOGS_APP /dev/vg_data/lv_logs

# Cambiar etiqueta a posteriori
e2label /dev/vg_data/lv_app NEW_LABEL      # Para ext4
xfs_admin -L NEW_LABEL /dev/vg_data/lv_logs # Para XFS (debe estar desmontado)
```

### Reparación y mantenimiento

```bash
# Verificar y reparar sistema de archivos ext4 (DEBE estar desmontado)
umount /mnt/data
fsck -y /dev/vg_data/lv_app

# Verificar y reparar sistema de archivos XFS (DEBE estar desmontado)
umount /mnt/logs
xfs_repair /dev/vg_data/lv_logs

# Ajustar porcentaje de bloques reservados para root en ext4 (por defecto es 5%, bajarlo a 1% libera espacio en discos grandes)
tune2fs -m 1 /dev/vg_data/lv_app
```

---

## 5. Gestión de Memoria Swap

Puedes crear espacio de intercambio usando una partición dedicada o un archivo (*swapfile*).

### Crear y activar un archivo Swap (Método universal)

```bash
# 1. Reservar archivo de 2 GB (usa dd si fallocate da problemas en XFS antiguos)
dd if=/dev/zero of=/swapfile bs=1M count=2048 status=progress

# 2. Asegurar permisos estrictos (obligatorio, solo root puede leer/escribir)
chmod 600 /swapfile

# 3. Formatear como área swap y activar
mkswap /swapfile
swapon /swapfile

# 4. Verificar memoria swap activa y prioridades
swapon --show
free -h
```

### Ajustar agresividad del Swap (`swappiness`)

```bash
# Ver valor actual (por defecto suele ser 60)
cat /proc/sys/vm/swappiness

# Configurar a 10 de forma persistente (preferir RAM antes que disco)
echo "vm.swappiness = 10" > /etc/sysctl.d/99-swap.conf
sysctl --system
```

---

## 6. Montaje Persistente (`/etc/fstab`)

Cualquier sistema de archivos o swap creado en el examen debe sobrevivir al reinicio mediante `/etc/fstab`. Usa **siempre `UUID=`** en lugar de `/dev/sdX` (los nombres de dispositivo pueden cambiar entre reinicios).

### Estructura de `/etc/fstab`

```text
# <dispositivo>                           <punto_montaje>  <tipo>  <opciones>        <dump> <pass>
UUID=a1b2c3d4-e5f6-7890-abcd-ef1234567890 /mnt/data        ext4    defaults          0      2
UUID=98765432-10fe-dcba-0987-654321fedcba /mnt/logs        xfs     defaults,noatime  0      0
/swapfile                                 none             swap    sw                0      0
```

### Validación obligatoria antes de reiniciar

{: .warning }
> **Regla de oro del LFCS:** Un error tipográfico en `/etc/fstab` dejará la máquina en *Emergency Mode* al reiniciar. Prueba siempre el montaje en caliente antes de dar el ejercicio por terminado.

```bash
# Crear directorio de montaje si no existe
mkdir -p /mnt/data

# Montar todo lo declarado en /etc/fstab y verificar errores
mount -a

# Recargar unidades de montaje de systemd (recomendado en RHEL 8/9 y Ubuntu 22.04+)
systemctl daemon-reload

# Confirmar que quedó montado correctamente
findmnt /mnt/data
```

---

## 7. Cuotas de Disco (`quotacheck`, `edquota`)

Limitar el espacio o número de archivos (inodos) que puede consumir un usuario o grupo en un punto de montaje `ext4`.

```bash
# 1. Añadir opciones 'usrquota,grpquota' en /etc/fstab para el punto de montaje (ej. /home)
# UUID=... /home ext4 defaults,usrquota,grpquota 0 2

# 2. Remontar para aplicar opciones
mount -o remount /home

# 3. Crear archivos de base de datos de cuotas y activarlas
quotacheck -cugm /home
quotaon -v /home

# 4. Editar límites (soft/hard) para un usuario específico
edquota -u devuser

# 5. Consultar informe general de cuotas del sistema
repquota -a
```

