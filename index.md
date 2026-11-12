---
title: Introducción
nav_order: 1
---

# Certificación LFCS (Linux Foundation Certified System Administrator)
{: .no_toc }

Repositorio técnico de apuntes, *cheat sheets* y procedimientos prácticos orientados a la administración de sistemas Linux en producción (**RHEL / Rocky Linux** y **Debian / Ubuntu**).
{: .fs-6 .fw-300 }

---

## Sobre el Examen Oficial

El **LFCS** es una certificación **100 % práctica basada en rendimiento** (*performance-based*). No existen preguntas tipo test: todas las tareas se resuelven directamente sobre terminales Linux reales en un entorno simulado.

| Parámetro | Detalle |
| :--- | :--- |
| **Duración** | 2 horas (120 minutos) |
| **Formato** | Terminal SSH en vivo con escenarios de resolución de problemas y configuración |
| **Distribuciones objetivo** | Familia Red Hat (RHEL / Rocky Linux) y familia Debian (Ubuntu) |
| **Puntuación mínima** | 67 % para aprobar |
| **Validez** | 3 años |

{: .note }
> **Enfoque de esta documentación:** Todos los comandos y configuraciones están diseñados para ser **persistentes tras reinicio** (un requisito obligatorio en la evaluación del examen) y listos para aplicar en entornos reales de producción.

---

## Mapa de Dominios y Estructura del Temario

Los módulos de esta guía cubren íntegramente los 5 dominios oficiales evaluados por The Linux Foundation:

| Módulo | Dominio Oficial LFCS | Peso | Contenido Clave |
| :--- | :--- | :---: | :--- |
| **[01. Operación de Sistemas]({{ site.baseurl }}/01-operacion-sistemas.html)** | *Operations Deployment* | **25 %** | `systemd`, procesos, prioridades, `journalctl`, cron/timers, `sysctl`, GRUB2. |
| **[02. Configuración de Redes]({{ site.baseurl }}/02-redes.html)** | *Networking* | **25 %** | `nmcli`, Netplan, `iproute2`, `ss`, DNS (`dig`), `chrony`, túneles SSH, VLANs. |
| **[03. Almacenamiento]({{ site.baseurl }}/03-almacenamiento.html)** | *Storage* | **20 %** | Particiones MBR/GPT, LVM completo, `ext4`/`xfs`, Swap, `/etc/fstab`, cuotas. |
| **[04. Comandos Esenciales]({{ site.baseurl }}/04-comandos-esenciales.html)** | *Essential Commands* | **20 %** | Flujos, `grep`, `sed`, `awk`, `find`, `tar`, `rsync`, SUID/SGID/Sticky, ACLs. |
| **[05. Usuarios y Seguridad]({{ site.baseurl }}/05-usuarios-seguridad.html)** | *Users and Groups* | **10 %** | Usuarios/grupos, PAM, `sudoers`, `firewalld`/`ufw`, SELinux y AppArmor. |

---

## Reglas de Oro para el Día del Examen

* **Persistencia obligatoria:** Verifica siempre que los servicios queden habilitados (`systemctl enable`), las reglas de firewall guardadas (`--permanent`) y los montajes declarados en `/etc/fstab` (validados con `mount -a`).
* **Uso de `man` y `--help`:** Tienes acceso completo a las páginas del manual local (`man -k <palabra_clave>` es tu mejor aliado si olvidas una sintaxis exacta).
* **Gestión del tiempo:** Si un ejercicio se atasca más de 8-10 minutos, márcalo, avanza al siguiente y vuelve al final.
