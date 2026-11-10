---
title: 05. Usuarios y Seguridad
nav_order: 6
---

# Usuarios, Privilegios y Hardening del Sistema
{: .no_toc }

Guía de referencia rápida para administración de cuentas, control de acceso obligatorio (MAC), cortafuegos y aseguramiento de servicios en RHEL/Rocky y Debian/Ubuntu.
{: .fs-6 .fw-300 }

## Tabla de contenidos
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 1. Gestión de Usuarios, Grupos y Contraseñas

### Creación y modificación de cuentas

| Acción | Comando | Detalle técnico |
| :--- | :--- | :--- |
| Crear usuario estándar | `useradd -m -s /bin/bash -c "Admin User" juan` | Crea `/home/juan` desde `/etc/skel` |
| Crear usuario de sistema (sin login) | `useradd -r -s /sbin/nologin appuser` | UID < 1000, sin directorio home |
| Añadir a grupos secundarios | `usermod -aG wheel,docker juan` | `-a` evita sobrescribir grupos actuales |
| Cambiar grupo primario | `usermod -g devops juan` | Modifica el GID principal en `/etc/passwd` |
| Bloquear / Desbloquear cuenta | `usermod -L juan` / `usermod -U juan` | Prefija `!` al hash en `/etc/shadow` |
| Borrar usuario y su home | `userdel -r juan` | Elimina `/home/juan` y spool de correo |
| Crear / Borrar grupo | `groupadd -g 2000 audit` / `groupdel audit` | Asigna GID específico si se indica `-g` |

### Políticas de caducidad (`chage`)
```bash
# Ver estado de expiración de un usuario
chage -l juan

# Forzar cambio de contraseña en el próximo inicio de sesión
chage -d 0 juan

# Configurar caducidad: mín 7 días, máx 90 días, aviso 14 días antes, bloqueo 5 días tras expirar
chage -m 7 -M 90 -W 14 -I 5 juan

# Establecer fecha exacta de expiración de la cuenta
chage -E 2027-12-31 juan
```

---

## 2. Privilegios (`sudo`), Límites y PAM

### Reglas en `/etc/sudoers.d/`
Nunca edites `/etc/sudoers` directamente. Crea ficheros modulares en `/etc/sudoers.d/`:

```bash
# Permitir a un grupo ejecutar comandos específicos sin contraseña
echo "%operadores ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/journalctl" > /etc/sudoers.d/operadores

# Asignar permisos estrictos al fichero y validar sintaxis (CRUCIAL)
chmod 0440 /etc/sudoers.d/operadores
visudo -cf /etc/sudoers.d/operadores
```

### Límites de recursos del sistema (`/etc/security/limits.conf`)
En el examen suelen pedir limitar procesos (*nproc*) o ficheros abiertos (*nofile*) para evitar *fork bombs* o saturación:

```bash
# Ver límites actuales de la sesión
ulimit -a

# Configurar límites persistentes en /etc/security/limits.d/90-custom.conf
cat << 'EOF' > /etc/security/limits.d/90-custom.conf
@desarrolladores   soft    nproc           200
@desarrolladores   hard    nproc           400
juan               hard    nofile          8192
EOF
```

---

## 3. Permisos Especiales, SUID/SGID, Sticky Bit y ACLs

### Bits especiales (`chmod`)

| Bit | Octal | Simbólico | Efecto en Archivos | Efecto en Directorios |
| :--- | :--- | :--- | :--- | :--- |
| **SUID** | `4xxx` | `u+s` | Se ejecuta con permisos del **dueño** del binario | No tiene efecto |
| **SGID** | `2xxx` | `g+s` | Se ejecuta con permisos del **grupo** del binario | Los archivos nuevos heredan el **grupo** del directorio |
| **Sticky** | `1xxx` | `+t` | No tiene efecto | Solo el dueño del archivo (o root) puede borrarlo (`/tmp`) |

```bash
# Ejemplo típico de examen: directorio compartido donde todos heredan grupo y nadie borra lo ajeno
mkdir /srv/compartido
chown root:devops /srv/compartido
chmod 3770 /srv/compartido   # 2000 (SGID) + 1000 (Sticky) + 0770 (rwxrwx---)

# Buscar todos los binarios SUID en el sistema (auditoría de seguridad)
find / -perm -4000 -type f 2>/dev/null
```

### Listas de Control de Acceso (`getfacl` / `setfacl`)
```bash
# Dar permisos de lectura/escritura a un usuario puntual sin cambiar dueño ni grupo
setfacl -m u:juan:rw /srv/informe.txt

# Aplicar ACL recursiva (-R) y por defecto (-d) para futuros archivos en un directorio
setfacl -R -m g:audit:rx /var/log/app/
setfacl -d -m g:audit:rx /var/log/app/

# Ver ACLs activas / Eliminar todas las ACLs de un recurso
getfacl /srv/informe.txt
setfacl -b /srv/informe.txt
```

---

## 4. Hardening de SSH (`/etc/ssh/sshd_config`)

```bash
# Parámetros habituales exigidos en auditorías y exámenes
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
AllowGroups wheel sshusers
```

```bash
# Validar sintaxis antes de recargar el demonio (si falla, no te quedas fuera)
sshd -t && systemctl reload sshd
```

{: .warning }
> **Atención si cambias el puerto SSH:** Si cambias `Port 22` por `Port 2222` en RHEL/Rocky con SELinux activo, `sshd` fallará al arrancar hasta que registres el puerto con `semanage port -a -t ssh_port_t -p tcp 2222` y lo abras en el firewall.

---

## 5. Cortafuegos: `firewalld` (RHEL) y `ufw` (Ubuntu)

### `firewalld` (CentOS / RHEL / Rocky / AlmaLinux)
Todas las reglas permanentes llevan `--permanent` y requieren `--reload` para aplicarse (o usa `--runtime-to-permanent` tras probarlas).

```bash
# Consultar estado, zonas activas y reglas actuales
firewall-cmd --state
firewall-cmd --get-active-zones
firewall-cmd --list-all

# Abrir un servicio predefinido o un puerto específico de forma permanente
firewall-cmd --zone=public --add-service=https --permanent
firewall-cmd --zone=public --add-port=8080/tcp --permanent

# Regla rica (Rich Rule): permitir SSH solo desde una subred específica
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.10.0/24" service name="ssh" accept' --permanent

# Redirección de puertos (Port Forwarding: 80 -> 8080)
firewall-cmd --zone=public --add-forward-port=port=80:proto=tcp:toport=8080 --permanent

# Aplicar cambios permanentes al runtime actual
firewall-cmd --reload
```

### `ufw` (Debian / Ubuntu)
```bash
# Activar cortafuegos y revisar reglas numeradas
ufw enable
ufw status verbose
ufw status numbered

# Políticas por defecto (denegar entrada, permitir salida)
ufw default deny incoming
ufw default allow outgoing

# Permitir puertos, servicios y subredes
ufw allow ssh
ufw allow 443/tcp
ufw allow from 10.0.0.0/8 to any port 22 proto tcp

# Eliminar una regla por su número
ufw delete 3
```

---

## 6. Control de Acceso Obligatorio: SELinux y AppArmor

### SELinux (RHEL / Rocky / CentOS)
```bash
# Ver modo actual (Enforcing, Permissive, Disabled) y cambiar en caliente
getenforce
setenforce 1          # 1 = Enforcing, 0 = Permissive (para depurar fallos)
# Persistente en /etc/selinux/config -> SELINUX=enforcing

# Ver contexto SELinux de archivos (-Z), procesos (-Z) y puertos (-l)
ls -Z /var/www/html/
ps auxZ | grep httpd
semanage port -l | grep http

# Corregir contexto de un directorio personalizado (ej. web fuera de /var/www)
semanage fcontext -a -t httpd_sys_content_t "/srv/web(/.*)?"
restorecon -Rv /srv/web

# Permitir un puerto no estándar para un servicio (ej. Apache en puerto 8081)
semanage port -a -t http_port_t -p tcp 8081

# Gestionar booleanos de SELinux (ej. permitir a Apache conectar por red o leer homes)
getsebool -a | grep httpd
setsebool -P httpd_can_network_connect on
setsebool -P httpd_enable_homedirs on
```

{: .note }
> **Diagnóstico rápido de bloqueos SELinux:** Si un servicio falla sin motivo aparente de permisos estándar (`chmod`/`chown`), busca denegaciones recientes con `ausearch -m AVC,USER_AVC -ts recent` o revisa `/var/log/audit/audit.log`.

### AppArmor (Ubuntu / Debian / SUSE)
```bash
# Ver estado de perfiles cargados (enforce vs complain)
aa-status

# Cambiar el modo de un perfil concreto (requiere paquete apparmor-utils)
aa-enforce /etc/apparmor.d/usr.sbin.nginx     # Modo bloqueo estricto
aa-complain /etc/apparmor.d/usr.sbin.nginx    # Modo aprendizaje/log sin bloquear

# Recargar perfiles tras editar manualmente un fichero en /etc/apparmor.d/
apparmor_parser -r /etc/apparmor.d/usr.sbin.nginx
```
