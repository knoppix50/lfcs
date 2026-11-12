---
title: 04. Comandos Esenciales y Manipulación de Archivos
nav_order: 5
---

# 04. Comandos Esenciales y Manipulación de Archivos
{: .no_toc }

Procesamiento de flujos de texto (`grep`, `awk`, `sed`), redirecciones, búsqueda avanzada (`find`), empaquetado y compresión (`tar`), enlaces simbólicos/duros y permisos avanzados (SUID, SGID, Sticky Bit y ACLs).
{: .fs-6 .fw-300 }

## Contenido
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 1. Redirecciones, Pipes y Control de Flujos

En Linux todo flujo estándar usa tres descriptores: **`0` (stdin)**, **`1` (stdout)** y **`2` (stderr)**.

| Operador | Efecto | Ejemplo práctico |
| :--- | :--- | :--- |
| `>` / `>>` | Sobrescribe / Añade `stdout` a un archivo | `ls -la > listado.txt` |
| `2>` / `2>>` | Redirige solo errores (`stderr`) | `find / -name "*.conf" 2> /dev/null` |
| `&>` (o `> file 2>&1`) | Redirige `stdout` y `stderr` al mismo archivo | `/opt/script.sh &> /var/log/script.log` |
| `\| tee` | Muestra salida en pantalla y la guarda a la vez | `echo "vm.swappiness=10" \| tee -a /etc/sysctl.conf` |
| `xargs` | Convierte `stdin` en argumentos para otro comando | `find /tmp -name "*.tmp" \| xargs rm -f` |

---

## 2. Procesamiento de Texto (`grep`, `sed`, `awk`, `cut`, `sort`)

### Búsqueda con expresiones regulares (`grep -E`)

```bash
# Buscar sin distinguir mayúsculas (-i), recursivo (-r), mostrando número de línea (-n)
grep -rin "error" /var/log/nginx/

# Limpiar archivos de configuración (mostrar líneas que NO sean comentarios # ni líneas vacías)
grep -Ev '^\s*#|^\s*\$' /etc/ssh/sshd_config

# Extraer solo el fragmento coincidente (-o), útil para capturar IPs en logs
grep -Eo '([0-9]{1,3}\.){3}[0-9]{1,3}' /var/log/secure | sort -u

# Mostrar 3 líneas antes (-B) y 3 después (-A) de cada coincidencia
grep -C 3 "CRITICAL" /var/log/syslog
```

### Edición en flujo (`sed`)

```bash
# Sustituir texto en pantalla (primer match por línea vs global con /g)
sed 's/http:/https:/g' urls.txt

# Editar archivo directamente en disco (-i) creando copia de seguridad (.bak)
sed -i.bak 's/^PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config

# Eliminar líneas que contengan un patrón específico (/d)
sed -i '/192\.168\.1\.99/d' /etc/hosts

# Imprimir exclusivamente un rango de líneas (ej. de la línea 10 a la 20)
sed -n '10,20p' /var/log/messages
```

### Extracción y filtrado por columnas (`awk` y `cut`)

```bash
# Imprimir usuario (\$1) y shell (\$7) usando ':' como separador (-F)
awk -F: '{print \$1, \$7}' /etc/passwd

# Filtrar usuarios con UID >= 1000 (usuarios normales en RHEL/Debian)
awk -F: '\$3 >= 1000 && \$1 != "nobody" {print \$1 " -> UID:" \$3}' /etc/passwd

# Sumar valores de una columna (ej. calcular consumo total de RSS en KB)
ps aux | awk '{sum += \$6} END {print "Total RAM (MB):", sum/1024}'

# Alternativa rápida con cut para campos delimitados simples
cut -d: -f1,6 /etc/passwd
```

---

## 3. Búsqueda Avanzada de Archivos (`find` y `locate`)

El comando `find` es una pieza central en el examen práctico para localizar archivos por criterios combinados y ejecutar acciones masivas sobre ellos.

```bash
# Buscar archivos (-type f) mayores de 50 MB en /var y ordenarlos por tamaño
find /var -type f -size +50M -exec ls -lh {} + 2>/dev/null

# Buscar archivos modificados en las últimas 24 horas (-mtime 0) o hace más de 30 días (+30)
find /etc -type f -mtime -1
find /var/log -name "*.gz" -mtime +30 -delete

# Buscar por propietario o grupo y copiar resultados a un directorio destino
find /home -user devuser -name "*.sh" -exec cp -p {} /backup/scripts/ \;

# Buscar archivos con permisos inseguros (ej. escribibles por cualquiera: 777 o o+w)
find /srv -type f -perm -o=w

# Buscar archivos con bit SUID activado (auditoría de seguridad)
find / -perm /4000 -type f 2>/dev/null
```

{: .note }
> **Diferencia entre `-exec ... \;` y `-exec ... +`:** Usar `\;` ejecuta el comando una vez por cada archivo encontrado. Usar `+` agrupa todos los archivos en una sola ejecución (mucho más rápido, similar a `xargs`).

---

## 4. Empaquetado, Compresión y Sincronización (`tar`, `rsync`)

### Operaciones habituales con `tar`

```bash
# Crear archivo comprimido con gzip (.tar.gz / .tgz)
tar -czvf backup_etc.tar.gz /etc

# Crear archivo comprimido con bzip2 (.tar.bz2) o xz (.tar.xz)
tar -cjvf backup_var.tar.bz2 /var/www
tar -cJvf backup_opt.tar.xz /opt

# Listar el contenido de un tar sin extraerlo (-t)
tar -tvf backup_etc.tar.gz

# Extraer en un directorio específico (-C)
mkdir -p /tmp/restore
tar -xzvf backup_etc.tar.gz -C /tmp/restore

# Preservar permisos y contextos SELinux al empaquetar
tar --selinux --acls -czpvf backup_completo.tar.gz /srv/app
```

### Sincronización eficiente con `rsync`

```bash
# Sincronización local conservando permisos, propietario, fechas y enlaces (-a)
rsync -avh --progress /srv/datos/ /mnt/backup/datos/

# Sincronizar eliminando en destino lo que ya no existe en origen (--delete)
rsync -avh --delete /srv/datos/ /mnt/backup/datos/

# Sincronización remota sobre SSH
rsync -avz -e "ssh -p 2222" /srv/datos/ admin@192.168.1.50:/backup/
```

---

## 5. Enlaces Duros (*Hard Links*) vs Simbólicos (*Soft Links*)

| Característica | Hard Link (`ln origen enlace`) | Symbolic Link (`ln -s origen enlace`) |
| :--- | :--- | :--- |
| **Inodo** | Comparte el **mismo inodo** que el original. | Tiene su **propio inodo** (apunta a la ruta). |
| **Si borras el original** | El dato sigue accesible en el hard link. | El enlace queda roto (*dangling link*). |
| **Directorios** | **No** permitido entre directorios. | **Sí** permitido. |
| **Entre particiones / FS** | **No** (solo dentro del mismo filesystem). | **Sí** (cruza particiones y discos). |

```bash
# Crear enlace simbólico y verificar a dónde apunta
ln -s /usr/local/bin/app_v2 /usr/local/bin/app
ls -li /usr/local/bin/app*
```

---

## 6. Permisos Estándar, Especiales y ACLs

### Permisos Especiales (`SUID`, `SGID`, `Sticky Bit`)

| Bit | Octal | Simbólico | Efecto en Archivos | Efecto en Directorios |
| :--- | :--- | :--- | :--- | :--- |
| **SUID** | `4000` | `u+s` | Se ejecuta con los permisos del **propietario** (ej. `/usr/bin/passwd`). | Sin efecto práctico estándar. |
| **SGID** | `2000` | `g+s` | Se ejecuta con los permisos del **grupo** propietario. | **Clave en trabajo colaborativo:** todo archivo creado dentro hereda el grupo del directorio. |
| **Sticky** | `1000` | `+t` | Sin efecto. | Solo el propietario del archivo (o `root`) puede borrarlo aunque todos tengan permiso `w` (ej. `/tmp`). |

```bash
# Configurar directorio compartido de equipo (SGID + Sticky Bit = 3770)
mkdir -p /srv/proyecto
chown root:desarrolladores /srv/proyecto
chmod 3770 /srv/proyecto
```

### Listas de Control de Acceso (`getfacl` y `setfacl`)

Permiten otorgar permisos granulares a usuarios o grupos específicos sin alterar el propietario ni el grupo base del archivo.

```bash
# Consultar ACLs actuales de un archivo o directorio
getfacl /srv/proyecto

# Dar permisos de lectura y escritura (rw) a un usuario concreto
setfacl -m u:auditor:rx /srv/proyecto/informe.log

# Aplicar ACL recursiva (-R) y por defecto (d:) para futuros archivos creados dentro del directorio
setfacl -R -m g:qa:rwx /srv/proyecto
setfacl -R -m d:g:qa:rwx /srv/proyecto

# Eliminar una entrada ACL específica (-x) o limpiar todas las ACLs (-b)
setfacl -x u:auditor /srv/proyecto/informe.log
setfacl -b /srv/proyecto/informe.log
```

{: .warning }
> **Indicador visual en `ls -l`:** Cuando un archivo o directorio tiene ACLs activas, aparece un símbolo **`+`** al final de su cadena de permisos (ejemplo: `-rw-rw-r--+`).

