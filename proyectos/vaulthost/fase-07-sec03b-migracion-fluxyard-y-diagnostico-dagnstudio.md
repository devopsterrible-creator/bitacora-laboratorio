# Bitácora — Sprint 2 / Ticket SEC-03B (parte Fluxyard) + Diagnóstico pre-migración Dagnstudio

> **Documento de bitácora de fase.** Registra la migración del segundo dominio (Fluxyard) a Cloudflare siguiendo el playbook validado con AGC, y documenta el diagnóstico completo read-only del tercer y último dominio (Dagnstudio) para su ejecución en la próxima sesión.
> Complementa al runbook oficial y a la bitácora `fase-06-sec03b-migracion-agc-cloudflare.md`.

| Campo | Valor |
|---|---|
| **Proyecto** | VaultHost VPS — hardening multi-tenant |
| **Sprint / Fase** | Sprint 2 (Hardening) — **Ticket SEC-03B (partes 2 y 3-diagnóstico)** |
| **Tipo de trabajo** | 🔍 Read-only + 🛠️ Modificador (Fluxyard completo · Dagnstudio solo diagnóstico) |
| **Responsable** | Oscar (owner, dev y administrador) |
| **Acompañamiento** | Claude (ingeniero de cabecera) |
| **Fecha** | 12-07-2026 |
| **Estado** | ✅ Fluxyard cerrado · 🔍 Dagnstudio diagnosticado (ejecución pendiente próxima sesión) |

---

## Control de versiones del documento

| Versión | Fecha | Cambio | Autor |
|---|---|---|---|
| 1.0 | 12-07-2026 | Bitácora Fluxyard + diagnóstico Dagnstudio | Oscar + Claude |

---

## 1. Objetivo de la fase

1. **Migrar `fluxyard-appsystem.cl` a Cloudflare** siguiendo el playbook validado con AGC, con la simplificación de que Fluxyard no requiere buzón operativo (proyecto en baja pendiente, sin uso real) — pero sí requiere protección de puertos porque expone la IP del VPS.
2. **Diagnosticar completamente `dagnstudio-vps.cl`** (el último dominio) en modo read-only, resolviendo las dudas de blast radius antes de su ejecución.

Ambos dominios usan la **cuenta Cloudflare de dagnstudio** (infra propia), a diferencia de AGC que usó cuenta propia del cliente.

---

## 2. Parte Fluxyard — Migración completa

### 2.1 — Contexto y decisiones

- **Cuenta Cloudflare:** dagnstudio (la misma que aloja vaulthost.cl y scm-pro.cl). Decisión: Fluxyard es proyecto 100% propio → cuenta propia de dagnstudio, no cuenta separada.
- **Sin buzón operativo:** el proyecto no tiene uso; se omite todo el sub-flujo de correo (Gmail + forwarder + Send-mail-as).
- **Estado del proyecto:** baja pendiente a futuro. La baja definitiva de la cuenta cPanel `fluxyardappsystm` + limpieza de app/API/archivos es un ticket separado. Hoy solo se protege la exposición de puertos.

### 2.2 — Paso 1: Inventario DNS (read-only)

Barrido dual (vista pública `@1.1.1.1` + autoritativa `@ns1.dagnstudio-vps.cl`).

**Hallazgos:**
- Bloques A y B idénticos → cero drift.
- AXFR denegado (`Transfer failed`) → PowerDNS seguro.
- Sin AAAA, sin CAA.
- MX auto-referenciado al apex (misma trampa que AGC).
- DKIM partido en 2 strings TXT (mismo tratamiento).
- Subdominios cPanel estándar completos (www, mail, cpanel, webmail, whm, autodiscover, autoconfig, cpcalendars, cpcontacts, ftp, webdisk).
- **Subdominio adicional detectado en CF:** `api.fluxyard-appsystem.cl` → servicio real (recibe datos de simulador IoT ESP32). Conservado en gris.

### 2.3 — Paso 4: Replicación de zona en Cloudflare

Como la cuenta CF de dagnstudio ya existía con 2FA, se saltó signup + seguridad. Directo a Add site.

**Fixes aplicados (idénticos al patrón AGC):**
- MX corregido: apex → `mail.fluxyard-appsystem.cl`
- CNAME `mail` → reemplazado por A record en gris
- 9 subdominios administrativos → toggle a gris
- DKIM validado por concatenación (fuente: cPanel de `fluxyardappsystm`)
- `api` subdominio → conservado en gris (protocolo IoT posiblemente no-HTTP)

**Auditoría por Claude in Chrome:** se usó el agente de navegador para auditar la zona en modo read-only. Reportó 28 registros, confirmó ausencia de las trampas críticas (MX al apex, mail como CNAME proxied, subdominios admin en naranja), y detectó el registro `api` (resuelto: es servicio IoT real). El agente también reportó el warning genérico de CF "origin IP partially exposed" en subdominios grises — **ignorado conscientemente** (es el trade-off esperado: no se puede proxiar correo/paneles).

**NS asignados por CF a Fluxyard:** `katja.ns.cloudflare.com` + `rudy.ns.cloudflare.com`

### 2.4 — Paso 5: Cambio de NS en NIC Chile

NS actualizados a `katja` + `rudy`. (Nota de comunicación: esta vez el submit fue confirmado correctamente, sin la ambigüedad "escrito vs guardado" del ticket AGC.)

### 2.5 — Paso 6: Validación de propagación

**Resultado:**
- 4/4 resolvers públicos sirviendo `katja` + `rudy`.
- Dashboard de CF: "Active" (zona protegida y proxeando).
- Apex resuelve a IPs de Cloudflare: `172.67.149.252` + `104.21.29.239` → **proxy naranja activo, IP oculta**.
- DKIM en nueva zona: idéntico al esperado.
- `mail` resuelve a `158.69.48.4` (VPS directo, correo intacto).

**Nota técnica — discrepancia de auditoría:** el agente de navegador (vía dns.google web) reportó momentáneamente el apex resolviendo a `158.69.48.4` en vez de IP de CF. La consulta desde Kali con `dig +short @1.1.1.1` confirmó las IPs correctas de Cloudflare. **Lección: `dig` directo a un resolver conocido supera a APIs web intermediarias durante ventanas de propagación.** La diferencia era rezago de cache entre edges de resolvers públicos, no error de config.

### 2.6 — Estado final Fluxyard

✅ **Migración completa y validada.** Web proxeada, IP oculta, correo preservado (aunque sin uso operativo), API IoT accesible directo por gris.

---

## 3. Parte Dagnstudio — Diagnóstico pre-migración (read-only)

Esta sección documenta el análisis de blast radius del último dominio, ejecutado en modo read-only. **La ejecución queda pendiente para la próxima sesión** por decisión de fatiga (evitar tocar infra propia crítica con ~5h de sesión acumuladas).

### 3.1 — Por qué Dagnstudio es especial

Tres razones que lo diferencian de AGC y Fluxyard:

1. **PowerDNS vive en este dominio.** `ns1/ns2.dagnstudio-vps.cl` han sido los NS autoritativos de los tres dominios. Al migrar dagnstudio, el PowerDNS del VPS pierde todo propósito.
2. **Buzón `admin@dagnstudio-vps.cl` es contacto comercial vivo** (no desechable como el de Fluxyard). Si el MX queda mal, Oscar pierde su propio correo operativo.
3. **Es infra propia sin red de rescate externa** — el operador y el afectado son la misma persona.

### 3.2 — Preocupación principal resuelta: ¿migrar el DNS rompe el acceso a WHM?

**Respuesta documentada (con fuente oficial): NO.**

- WHM se accede por puerto 2087 vía **hostname del servidor O dirección IP directa** (`https://158.69.48.4:2087`). El acceso por IP no depende de ningún DNS.
- Fuente oficial cPanel: WHM escucha en 2087 precisamente para que el tráfico web viva en 80/443; WHM está arquitectónicamente separado del DNS del dominio.
- Oscar ya accede a WHM vía `https://10.8.0.1:2087` sobre WireGuard (por IP), 100% independiente del DNS de dagnstudio-vps.cl.

### 3.3 — Separación de capas (la confusión desarmada)

El nombre `dagnstudio-vps` aparece en múltiples capas que son independientes:

| Capa | Qué es | ¿Se afecta al migrar DNS a CF? |
|---|---|---|
| Hostname del servidor | `vps-7a76ae17.vps.ovh.ca` (dominio de OVH, NO dagnstudio) | ❌ NO se toca |
| NS del VPS (`ns1/ns2.dagnstudio-vps.cl`) | PowerDNS del servidor | ⚠️ Pierden rol, pero no afecta acceso |
| Cuenta cPanel `dagnstudiovps` | Hosting con dominio dagnstudio-vps.cl | ⚠️ Su web queda proxeada, cuenta intacta |
| Zona DNS de dagnstudio-vps.cl | Registros A/MX/TXT | ✅ Esto es lo único que migra |

### 3.4 — Verificación read-only ejecutada (Paso 0)

| Check | Resultado | Conclusión |
|---|---|---|
| ¿AGC usa NS internos? | `damon/emerie.ns.cloudflare.com` | ✅ Ya no depende de PowerDNS |
| ¿Fluxyard usa NS internos? | `katja/rudy.ns.cloudflare.com` | ✅ Ya no depende de PowerDNS |
| IP de ns1/ns2.dagnstudio-vps.cl | Ambos → 158.69.48.4 | Esperado (apuntan al VPS) |
| Zona dagnstudio actual | NS: PowerDNS · A: 158.69.48.4 · MX: `0 dagnstudio-vps.cl` (auto-ref) | Aún no migrado, misma trampa MX |
| **Hostname del servidor** | **`vps-7a76ae17.vps.ovh.ca`** | ✅ **NO usa dagnstudio-vps.cl** — sin complicación extra |

### 3.5 — Conclusión del diagnóstico

**Dagnstudio es ejecutable con seguridad.** Blast radius completamente mapeado:

- ✅ Ningún dominio externo depende de los NS internos (orden de migración cumplió su función).
- ✅ Hostname del servidor independiente (OVH) → migrar el DNS no toca el acceso al servidor.
- ✅ Acceso WHM garantizado por IP/VPN sin depender del DNS.
- ✅ No hay bombas escondidas de infraestructura.

**Matices a resolver en la ejecución:**
1. **Owner de la cuenta CF de dagnstudio:** verificar qué email es el owner (My Profile → Email). Si es un buzón en dagnstudio-vps.cl, agregar paracaídas de recuperación (segundo admin con Gmail externo, ej. devops.terrible@gmail.com) ANTES de migrar, para no quedar bloqueado durante la ventana de propagación.
2. **Buzón `admin@dagnstudio-vps.cl`:** preservar con MX apuntando a `mail.dagnstudio-vps.cl` en gris → VPS.
3. **PowerDNS post-migración:** evaluar apagarlo (queda huérfano). Ticket derivado, no urgente.
4. **Cuenta cPanel redundante `dagnstudiovps`:** consolidación para liberar slot de licencia. Ticket SEPARADO, no mezclar con esta migración.

---

## 4. Plan de ejecución de Dagnstudio (para próxima sesión)

```
Paso 0.5 — Blindaje del owner CF (PRE-requisito de seguridad)
    - Verificar email owner de la cuenta CF de dagnstudio
    - Si depende del VPS: agregar 2do admin externo (Gmail) como paracaídas
    - Probar login con la vía paracaídas antes de continuar

Paso 1 — Inventario DNS de dagnstudio-vps.cl (read-only, dual view)

Paso 4 — Replicar zona en cuenta CF de dagnstudio
    - Fix MX: apex → mail.dagnstudio-vps.cl (gris)
    - Subdominios admin → gris
    - @ y www → naranja
    - DKIM concatenado (fuente: cPanel)
    - ⚠️ Preservar buzón admin@ (MX gris al VPS)

Paso 5 — Cambio NS en NIC Chile

Paso 6 — Validación propagación + tests
    - Web + SSL
    - Correo admin@ entrante y saliente (SPF/DKIM/DMARC pass)

Post — Evaluar apagado de PowerDNS (ticket derivado)
```

---

## 5. Estado global del ticket SEC-03B

```
[✅] Parte 1 — AGC (agc-conectachile.cl)          — bitácora fase-06
[✅] Parte 2 — Fluxyard (fluxyard-appsystem.cl)   — bitácora fase-07 (este doc)
[🔍] Parte 3 — Dagnstudio (dagnstudio-vps.cl)     — diagnosticado, ejecución pendiente
[⏭️] Post — retomar Tarea 2 (candado 80/443 a Cloudflare IPs)
```

---

## 6. Lecciones aprendidas

### 6.1 — Técnicas

- **El acceso a WHM por IP es independiente del DNS del dominio.** Migrar la zona DNS nunca rompe el acceso administrativo por IP/VPN. Fuente oficial cPanel confirmada.
- **El hostname del servidor y el dominio hospedado son capas distintas** aunque compartan nombre parcialmente. Verificar `hostname -f` antes de asumir dependencias.
- **`dig` directo a resolver conocido > APIs web intermediarias** durante ventanas de propagación. Las herramientas web pueden reportar cache desactualizado.
- **El orden de migración (AGC→Fluxyard→Dagnstudio) neutralizó el riesgo de cascada de NS.** Al migrar dagnstudio, ya nadie depende de sus NS internos.

### 6.2 — Metodológicas

- **Frenar ante infra propia crítica con fatiga acumulada es decisión de ingeniero senior, no debilidad.** El instinto de Oscar de no ejecutar dagnstudio a las ~5h de sesión fue correcto.
- **Diagnóstico read-only completo antes de tocar = ejecución sin sorpresas después.** El Paso 0 mapeó todo el blast radius sin modificar nada.
- **Consultar documentación oficial adicional al contexto del proyecto** (regla de Oscar) evitó responder de memoria sobre el acceso a WHM — se confirmó con fuente cPanel.

### 6.3 — De colaboración

- **El agente de navegador (Claude in Chrome) como auditor read-only funciona bien** para validación de zona DNS, pero sus resultados vía APIs web deben contrastarse con `dig` directo. Útil como segunda opinión, no como fuente única de verdad.

---

## 7. English practice — glosario de la fase

| English | Español | Uso en esta fase |
|---|---|---|
| Blast radius | Radio de impacto | Qué se rompe si algo sale mal en la migración de dagnstudio |
| Hostname | Nombre de host | `vps-7a76ae17.vps.ovh.ca` — el nombre de la máquina |
| Out-of-band access | Acceso fuera de banda | Entrar a WHM por IP/VPN sin depender del DNS |
| Circular dependency | Dependencia circular | El owner CF cuyo email vive en el dominio a migrar |
| Parachute / failsafe | Paracaídas / a prueba de fallos | 2do admin externo en la cuenta CF |
| Orphaned nameserver | Nameserver huérfano | ns1/ns2.dagnstudio tras migrar todos los dominios |
| Staging / diagnosis | Diagnóstico previo | El Paso 0 read-only antes de ejecutar |

> *Practice:* "Map the blast radius before you touch production infrastructure — a read-only diagnosis that costs ten minutes can save you from an outage that costs a day."
> *Práctica:* "Mapea el radio de impacto antes de tocar infraestructura en producción — un diagnóstico de solo lectura que cuesta diez minutos puede salvarte de una caída que cuesta un día."

---

## 8. Definition of Done

**Fluxyard (completo):**
- [x] Inventario DNS dual
- [x] Zona replicada en cuenta CF de dagnstudio
- [x] Fix MX + CNAME mail + subdominios grises + DKIM
- [x] Auditoría por Claude in Chrome (28 registros validados)
- [x] `api` subdominio IoT conservado en gris
- [x] NS cambiados en NIC Chile (katja + rudy)
- [x] Propagación validada (4/4 resolvers + dashboard Active)
- [x] Proxy naranja activo (172.67.149.252 + 104.21.29.239)
- [x] DKIM y mail validados en nueva zona

**Dagnstudio (diagnóstico):**
- [x] Blast radius mapeado
- [x] Confirmado: nadie depende de NS internos
- [x] Confirmado: hostname independiente (OVH)
- [x] Confirmado: acceso WHM por IP garantizado
- [x] Plan de ejecución documentado para próxima sesión
- [ ] ⏭️ Ejecución (próxima sesión)

---

*Fin de la bitácora — Sprint 2 / Ticket SEC-03B partes Fluxyard + diagnóstico Dagnstudio · VaultHost.cl · 12-07-2026*
