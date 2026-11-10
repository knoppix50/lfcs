---
title: 05. Usuarios y Hardening
nav_order: 6
---

# Usuarios, Permisos y Hardening del Sistema
{: .no_toc }

Guía rápida de administración de cuentas, privilegios y aseguramiento básico en Linux.
{: .fs-6 .fw-300 }

## Tabla de contenidos
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Gestión de Usuarios y Grupos

| Acción | Comando | Notas clave |
| :--- | :--- | :--- |
| Crear usuario con home y bash | `useradd -m -s /bin/bash usuario` | `-m` crea `/home/usuario` |
| Añadir a grupo secundario | `usermod -aG wheel usuario` | **Siempre** usar `-aG` para no borrar grupos previos |
| Bloquear / Desbloquear cuenta | `usermod -L usuario` / `usermod -U usuario` | Añade `!` al hash en `/etc/shadow` |
| Caducidad de contraseña | `chage -M 90 -W 7 usuario` | Expira a los 90 días, avisa 7 antes |

{: .warning }
> **Cuidado en examen:** Si ejecutas `usermod -G grupo usuario` sin la `-a` (*append*), eliminarás al usuario de todos sus grupos actuales, incluido `sudo` o `wheel`.

---

## Privilegios (`sudo`) y Hardening SSH

### Configuración segura de `sudoers`
Nunca edites `/etc/sudoers` directamente con `vi` o `nano`. Crea archivos independientes en `/etc/sudoers.d/` validando la sintaxis:

```bash
# Crear regla sin contraseña para un grupo de administradores
echo "%sysadmins ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/sysadmins

# Validar sintaxis antes de salir (evita bloquear el sistema)
visudo -cf /etc/sudoers.d/sysadmins
```

### Aseguramiento básico de SSH (`/etc/ssh/sshd_config`)
```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers admin_user
```

{: .note }
> **Verificación antes de reiniciar SSH:** Ejecuta siempre `sshd -t` para comprobar que no hay errores de sintaxis antes de lanzar `systemctl reload sshd`.
