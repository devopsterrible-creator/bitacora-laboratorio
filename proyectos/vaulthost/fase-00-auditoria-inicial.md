# Fase 00 — Auditoría inicial del VPS

| Campo | Valor |
|---|---|
| Proyecto | VaultHost.cl |
| Fase / Ticket | Sprint 1 · Pre-flight |
| Fecha | 24-05-2026 |
| Tecnologías | AlmaLinux, cPanel/WHM, Apache, Node |
| Estado | ✅ Completada |

## 1. Objetivo
Fotografiar el estado real del VPS antes de tocar nada, para planificar el
despliegue con datos y no con suposiciones.

## 2. Contexto y prerequisitos
Acceso root por SSH (PuTTY) al VPS. Solo comandos de lectura.

## 3. Procedimiento ejecutado

### Paso 1 — Identificar SO y usuario
- **Qué:** versión del SO y con qué usuario operamos.
- **Comando:** `cat /etc/almalinux-release ; uname -r ; whoami ; id`
- **Por qué:** confirma que el kernel es AlmaLinux estándar (no CloudLinux), lo que descarta el "Node.js Selector" de CloudLinux.
- **Resultado:** AlmaLinux 9.7, kernel estándar, usuario root.

### Paso 2 — Listar cuentas cPanel y dominios
- **Qué:** ver todas las cuentas del servidor.
- **Comando:** `ls -1 /var/cpanel/users/ ; whmapi1 listaccts`
- **Por qué:** identificar la cuenta de VaultHost y no tocar las demás.
- **Resultado:** 5 cuentas. La de VaultHost es `vaulthostweb` → vaulthost.cl.

### Paso 3 — Verificar motor Node de cPanel (Passenger)
- **Qué:** ver si existe ea-nodejs y el módulo Passenger.
- **Comando:** `ls -d /opt/cpanel/ea-nodejs*/ ; /usr/sbin/httpd -M | grep -i passenger`
- **Por qué:** son el requisito de la Opción A elegida.
- **Resultado:** ❌ ninguno instalado → se vuelve el primer ticket del Sprint 1.

### Paso 4 — Apache, puertos y recursos
- **Comando:** `/usr/sbin/httpd -v ; ss -tlnp | grep -E ':80|:443' ; df -h / ; free -h`
- **Por qué:** confirmar el servidor web y que hay recursos para desplegar.
- **Resultado:** Apache 2.4.67 en 80/443; 85 GB libres; 9.2 GB RAM disponible.

## 4. Verificación / evidencia
Salida de los comandos guardada (snapshot del 24-05-2026).

## 5. Problemas y solución
| Problema | Causa | Solución |
|---|---|---|
| Falta ea-nodejs y Passenger | El servidor nunca fue aprovisionado para Node | Instalar vía EasyApache 4 (Fase 01) |

## 6. Decisiones (mini-ADR)
Ver ADR-001: la Opción A se mantiene pese a faltar el motor — se aprovisiona, no se cambia de método.

## 7. Lecciones aprendidas
"Discovery first, build second": una auditoría read-only de 6 comandos evitó
asumir Nginx o un stack inexistente. **Medir antes de planificar.**

## 8. Referencias
- runbook.md (este proyecto)
- ADR-001
