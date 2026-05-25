# Bitácora — Sprint 1 / Ticket 2: Provision del stack Node + Passenger

> **Documento de bitácora de fase.** Registra lo ejecutado, los problemas, las decisiones y las lecciones de esta fase del proyecto VaultHost.cl.
> Complementa al runbook oficial (*VaultHost — Arquitectura y Runbook de Despliegue*); cualquier corrección al runbook detectada aquí se anota en §7.

| Campo | Valor |
|---|---|
| **Proyecto** | VaultHost VPS — sitio comercial vaulthost.cl |
| **Sprint / Fase** | Sprint 1 (Despliegue) — **Ticket 2**: provision del stack Node + Passenger |
| **Tipo de trabajo** | 🛠️ Modificador (primer cambio real en el servidor) |
| **Responsable** | Oscar (owner, dev y administrador) |
| **Acompañamiento** | Claude (ingeniero de cabecera) |
| **Fecha** | 25-05-2026 |
| **Estado** | ✅ Ticket 2 cerrado · ⏭️ Application Manager + registro pendiente (Ticket 3) |
| **Método de despliegue** | Opción A — cPanel Application Manager + Phusion Passenger |

---

## Control de versiones del documento

| Versión | Fecha | Cambio | Autor |
|---|---|---|---|
| 1.0 | 25-05-2026 | Bitácora inicial del Ticket 2 (provision) | Oscar + Claude |

---

## 1. Objetivo de la fase

Ejecutar el **primer paso modificador** del proyecto: provisionar el stack Node.js 22 + Phusion Passenger + companion modules vía EasyApache 4, validar la instalación, y confirmar el checkpoint del "Node fantasma" identificado en el Ticket 1.

Principio metodológico reforzado: *explicar la operación antes de ejecutar; el usuario modifica, el ingeniero guía.* Este fue el primer 🛠️ real del proyecto, tratado con el respeto y cuidado que merece una operación que cambia el estado del servidor.

---

## 2. Punto de partida

| Recurso | Valor |
|---|---|
| Estado previo (Ticket 1) | Node y Passenger confirmados disponibles en repo EA4-c9, pero NO instalados |
| Lista de compras | `ea-nodejs22` (22.22.3) + `ea-apache24-mod-passenger` (6.1.3) + `ea-apache24-mod_env` (2.4.67) |
| Apache | Corriendo desde 2026-05-20 04:09:05 UTC (sin reiniciar desde antes del Ticket 1) |
| Cuentas cPanel | 5 cuentas activas (agcapp, dagnstudiovps, fluxyardappsystm, scmpro, vaulthostweb) |

---

## 3. Pasos ejecutados (QUÉ / COMANDO / POR QUÉ / RESULTADO)

### Pre-fase — Briefing de la operación (awareness antes de modificar)

**QUÉ:** explicación de qué hace un Provision de EasyApache y sus consecuencias.
**POR QUÉ:** el Provision reinicia Apache brevemente (~30 seg), interrumpiendo las otras 4 cuentas cPanel. Awareness profesional: briefing a stakeholders antes de modificar infraestructura compartida.
**ACCIÓN:** recomendación de ejecutar en horario de bajo tráfico.

### Paso 1 — Provision vía EasyApache GUI 🛠️

**1.1 — Navegación a Customize**
- **RUTA:** WHM → buscar "EasyApache 4" → botón "Customize" del perfil activo
- **RESULTADO:** interfaz de Customize cargada, con pestañas "Additional Packages" y "Apache Modules"

**1.2 — Marcar nodejs22**
- **QUÉ:** pestaña "Additional Packages" → buscar `nodejs` → toggle `ea-nodejs22`
- **RESULTADO:** toggle azul "Install" para ea-nodejs22 (22.22.3) ✅

**1.3 — Marcar mod-passenger**
- **QUÉ:** buscar `passenger` → toggle `ea-apache24-mod-passenger`
- **NOTA:** el toggle apareció bajo la sección "Ruby via Passenger" (nombre confuso pero correcto, ya documentado en Ticket 1)
- **RESULTADO:** toggle azul "Install" para mod-passenger (6.1.3) ✅

**1.4 — Marcar mod_env**
- **QUÉ:** buscar `mod_env` → toggle `ea-apache24-mod_env`
- **RESULTADO:** toggle azul "Install" para mod_env (2.4.67) ✅

**1.5 — Review (pre-provision)**
- **QUÉ:** botón "Review" al final de Customize
- **RESULTADO:** pantalla de Review mostró:
  - **Packages to install:** nodejs22, mod-passenger, mod_env, passenger-runtime, ruby, rubygems (y dependencias Ruby que Passenger necesita) ✅
  - **Packages to upgrade:** ninguno ✅
  - **Packages to uninstall:** ninguno ✅
  - **Packages not affected:** lista larga de paquetes existentes (php81, php82, apache24, mod_security2, etc.) intactos ✅
- **CONFIRMACIÓN:** la lista de cambios era limpia y correcta — solo los 3 paquetes solicitados + dependencias esperadas.

**1.6 — Provision (ejecución)**
- **QUÉ:** botón azul "Provision"
- **ACCIÓN:** clic en Provision → barra de progreso (duración ~3-10 minutos)
- **RESULTADO:** provision completado (confirmado después por validación, ver Paso 2)

**1.7 — Pantalla post-provision (confusión momentánea)**
- **QUÉ:** después del Provision, Oscar vio una pantalla que decía "There are no changes needed to provision based on the profile of your selection."
- **CAUSA (explicada después):** el provision ya había completado, y el perfil estaba sincronizado con el sistema — el mensaje era correcto *post-facto* pero confuso en el flujo.
- **ACCIÓN:** proceder a validación terminal para confirmar la verdad del sistema (ver Problema P2).

### Paso 2 — Validación post-provision ✅ (solo-lectura)

```bash
# ¿Passenger cargado en Apache?
/usr/sbin/httpd -M | grep -i passenger
# RESULTADO: passenger_module (shared) ✅

# ¿ea-nodejs22 instalado?
ls -d /opt/cpanel/ea-nodejs22/
# RESULTADO: /opt/cpanel/ea-nodejs22/ ✅

# ¿Qué versión reporta?
/opt/cpanel/ea-nodejs22/bin/node -v
# RESULTADO: v22.22.3 ✅

# ¿Passenger runtime instalado?
rpm -qa | grep -i passenger
# RESULTADO: 
#   ea-passenger-runtime-1.0-2.10.1.cpanel.x86_64 ✅
#   ea-apache24-mod-passenger-6.1.3-1.2.1.cpanel.x86_64 ✅

# ¿Las 5 cuentas cPanel intactas?
ls -1 /var/cpanel/users/
# RESULTADO: agcapp, dagnstudiovps, fluxyardappsystm, scmpro, vaulthostweb ✅

# ¿Apache respondiendo normalmente?
systemctl status httpd | grep -i "active (running) since"
# RESULTADO: Active: active (running) since Mon 2026-05-25 00:46:08 UTC; 19min ago ✅
```

**CONCLUSIÓN:** Apache se reinició a las 00:46:08 UTC (19 minutos antes de la validación) → confirma que el Provision SÍ ocurrió y fue exitoso.

### Paso 3 — Checkpoint del "Node fantasma" ✅

```bash
# ¿Se instaló un nodejs de sistema (el de Passenger)?
which node 2>/dev/null && node -v
# RESULTADO: /bin/node → v20.20.2 ✅

# ¿Nuestro ea-nodejs22 en su ruta?
/opt/cpanel/ea-nodejs22/bin/node -v
# RESULTADO: v22.22.3 ✅
```

**CONFIRMACIÓN del escenario predicho en Ticket 1:** DOS Node en el sistema:
1. **Node de sistema:** `/bin/node` → v20.20.2 (NodeSource, dependencia de Passenger, lo usa Passenger internamente)
2. **Node de cPanel:** `/opt/cpanel/ea-nodejs22/bin/node` → v22.22.3 (el que queremos para la app Next.js)

**Checkpoint para Ticket 3 (Application Manager):** al registrar la app, confirmar que Application Manager apunte a `/opt/cpanel/ea-nodejs22/bin/node`, no al de sistema.

---

## 4. Problemas encontrados y cómo se resolvieron

### P1 — Confusión inicial entre pasos 🛠️ (modificador) y ✅ (validación)
- **Síntoma:** Oscar corrió los comandos de validación (Paso 2 y 3) *antes* de hacer el Paso 1 (Provision GUI), esperando que los comandos instalaran los paquetes.
- **Causa:** distinción no clara entre comandos de lectura (validación) vs. operación modificadora (Provision vía GUI).
- **Solución:** clarificación explícita de la diferencia:
  - 🛠️ **Modificador:** cambia el estado del servidor (en este caso, el Provision vía GUI de EasyApache).
  - ✅ **Validación:** solo lee el estado (comandos `ls`, `rpm -qa`, `httpd -M`, `systemctl status`). No instalan nada.
- **Lección:** en un flujo de deploy, los pasos van en orden: primero el 🛠️, después el ✅. La validación confirma que el modificador funcionó, no lo ejecuta.

### P2 — Mensaje "no changes needed" post-Provision generó confusión
- **Síntoma:** después de hacer Provision, Oscar vio una pantalla que decía "There are no changes needed to provision based on the profile of your selection" y preguntó si debía aplicar los comandos de validación.
- **Causa:** el Provision había completado exitosamente, y el perfil de EasyApache ya estaba sincronizado con el sistema. El mensaje era *correcto* en ese momento (no había más cambios pendientes), pero confuso en el flujo porque parecía indicar que nada había pasado.
- **Solución:** confiar en **la validación terminal como fuente de verdad**, no en los mensajes GUI post-facto. Los comandos de validación mostraron que todo estaba instalado correctamente: Passenger cargado, ea-nodejs22 en `/opt/cpanel/`, Apache reiniciado a las 00:46 UTC.
- **Lección reforzada:** *el terminal nunca miente.* Los mensajes de GUI pueden ser ambiguos o depender de estados transitorios; la validación vía comandos de sistema (rpm, ls, systemctl, httpd -M) es la verdad absoluta del servidor.

---

## 5. Decisiones tomadas

No se tomaron decisiones nuevas en este ticket. El Ticket 2 ejecutó las decisiones ya tomadas en el Ticket 1:
- **D1 (Ticket 1):** usar `ea-nodejs22` en vez de `ea-nodejs20` → ejecutado ✅
- **D2 (Ticket 1):** nombre correcto del paquete Passenger → ejecutado ✅
- **D3 (Ticket 1):** camino estándar (Application Manager + Passenger, no CloudLinux Selector) → confirmado ✅

---

## 6. Lecciones aprendidas

- **Distinguir claramente 🛠️ (modificador) de ✅ (validación) en flujos de deploy.** Los pasos modificadores cambian el sistema; las validaciones solo leen. Van en ese orden.
- **En GUI workflows (como EasyApache Customize), todos los pasos de la secuencia importan:** toggle → Review → Provision. Saltarse o malinterpretar uno rompe el flujo.
- **El terminal es la única fuente de verdad para el estado del servidor.** Los mensajes de GUI (especialmente los post-operación) pueden ser ambiguos. Validar siempre vía comandos de sistema directos.
- **El primer 🛠️ de un proyecto se trata con respeto:** awareness de impacto (otras cuentas cPanel), briefing claro, validación exhaustiva post-ejecución. Este hábito construye confiabilidad.
- **Checkpoint de runtime se cumplió:** el "Node fantasma" predicho en Ticket 1 se confirmó empíricamente. Ahora el siguiente ticket debe verificar que Application Manager apunte al Node correcto.

---

## 7. English practice — glosario de la fase

| English | Español | Uso en esta fase |
|---|---|---|
| Modifier step | Paso modificador | La operación de Provision que cambió el servidor |
| Validation step | Paso de validación | Los comandos read-only que confirmaron el éxito |
| Provision | Provisionar | Instalar/actualizar el perfil de EasyApache |
| Stakeholders | Partes afectadas | Las otras cuentas cPanel impactadas por el reinicio de Apache |
| Source of truth | Fuente de verdad | Los comandos de terminal como verdad del sistema |
| Briefing | Briefing / charla previa | Explicar la operación antes de ejecutarla |

> *Practice:* "The terminal is the single source of truth for server state — GUI messages can be ambiguous, but validation commands don't lie."
> *Práctica:* "El terminal es la única fuente de verdad para el estado del servidor — los mensajes GUI pueden ser ambiguos, pero los comandos de validación no mienten."

---

## 8. Estado final — Definition of Done del Ticket 2

- [x] Provision ejecutado vía EasyApache GUI (Customize → Review → Provision)
- [x] Passenger module cargado en Apache (`passenger_module (shared)`)
- [x] ea-nodejs22 (v22.22.3) instalado en `/opt/cpanel/ea-nodejs22/`
- [x] ea-passenger-runtime (1.0) + ea-apache24-mod-passenger (6.1.3) instalados
- [x] Apache reiniciado el 2026-05-25 a las 00:46:08 UTC
- [x] Checkpoint "Node fantasma" confirmado: dos Node en sistema (sistema v20.20.2 + cPanel v22.22.3)
- [x] Las 5 cuentas cPanel intactas y sin afectación permanente

---

## 9. Próximos pasos — Ticket 3

**Habilitar Application Manager y registrar la app.**

Pasos previstos:
1. Habilitar feature "Application Manager" en WHM para la cuenta `vaulthostweb`
2. Clonar el código del repo `github.com/devopsterrible-creator/vaulthost-web` a `/home/vaulthostweb/vaulthost-web`
3. Crear `.env.production` con credenciales de producción (MercadoPago, BASE_URL, WhatsApp real `56971456955`)
4. Confirmar `next.config.js` tiene `output: 'standalone'`
5. Build de producción: `npm ci && npm run build`
6. Registrar app en Application Manager (app root, startup file `.next/standalone/server.js`, Node version selector)
7. **Checkpoint crítico:** confirmar que Application Manager apuntó a `/opt/cpanel/ea-nodejs22/bin/node`, no al node de sistema
8. Validación: acceder a `https://vaulthost.cl` y verificar que la app responde

Tickets posteriores del Sprint 1 (después del Ticket 3):
- AutoSSL + redirección HTTP→HTTPS
- MercadoPago a producción (webhook y back_urls)
- Pruebas end-to-end (checkout + voucher + webhook)

---

*Fin de la bitácora — Sprint 1 / Ticket 2 · VaultHost.cl*
