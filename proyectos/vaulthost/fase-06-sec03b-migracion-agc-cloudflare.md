# Bitácora — Sprint 2 / Ticket SEC-03B (parte AGC): Migración de agc-conectachile.cl a Cloudflare

> **Documento de bitácora de fase.** Registra la migración completa del dominio del primer cliente activo (AGC) desde PowerDNS (VPS) hacia Cloudflare, incluyendo creación de buzón operativo, cuenta CF a nombre del cliente, replicación de zona y switch de NS en NIC Chile — sin downtime.
> Complementa al runbook oficial (*VaultHost — Arquitectura y Runbook de Despliegue*) y a las bitácoras previas de Sprint 2 (`fase-04`, `fase-05`).

| Campo | Valor |
|---|---|
| **Proyecto** | VaultHost VPS — hardening multi-tenant |
| **Sprint / Fase** | Sprint 2 (Hardening) — **Ticket SEC-03B (parte 1 de 3)**: migración AGC |
| **Tipo de trabajo** | 🔍 Read-only + 🛠️ Modificador (DNS externo + panel Cloudflare + NIC Chile) |
| **Responsable** | Oscar (owner, dev y administrador) |
| **Acompañamiento** | Claude (ingeniero de cabecera) |
| **Fecha** | 12-07-2026 |
| **Estado** | ✅ AGC cerrado · ⏭️ Fluxyard pendiente (parte 2) · ⏭️ Dagnstudio pendiente (parte 3) |
| **Duración de la sesión** | ~4 horas efectivas (con múltiples checkpoints de validación) |

---

## Control de versiones del documento

| Versión | Fecha | Cambio | Autor |
|---|---|---|---|
| 1.0 | 12-07-2026 | Bitácora inicial del ticket SEC-03B / AGC | Oscar + Claude |

---

## 1. Objetivo de la fase

Migrar completamente el dominio del primer cliente activo (**AGC — agc-conectachile.cl**) desde el DNS interno del VPS (PowerDNS `ns1/ns2.dagnstudio-vps.cl`) hacia Cloudflare, cumpliendo simultáneamente cuatro objetivos:

1. **Ocultar la IP real del VPS** detrás del proxy naranja de Cloudflare para la web principal.
2. **Preservar el correo del dominio** sin un solo minuto de downtime (SMTP + IMAP + webmail).
3. **Crear infraestructura administrativa operativa** para la primera vez: buzón real de soporte + Gmail como cliente IMAP + Send-mail-as (replicando el patrón validado con scm-pro).
4. **Cuenta Cloudflare a nombre de AGC como cliente**, no compartida con la cuenta comercial de DagnStudio o VaultHost (decisión arquitectónica de soberanía por cliente).

Este ticket forma parte del bloque **SEC-03B** más amplio, que involucra migrar los tres dominios delegados a PowerDNS (AGC, Fluxyard, Dagnstudio) antes de poder cerrar la Tarea 2 pendiente del Sprint 2 (candado 80/443 al ipset `cloudflare-ips`).

---

## 2. Punto de partida

Contexto heredado de las bitácoras `fase-05-ejecucion-wireguard-y-tarea2.md` y `fase-05-reflexion-fase-3b-parte-1.md`:

| Recurso | Estado inicial |
|---|---|
| DNS de AGC | Delegado a `ns1/ns2.dagnstudio-vps.cl` (PowerDNS del VPS) — **DNS SPOF crítico** |
| IP de origen | `158.69.48.4` — expuesta directamente al mundo |
| Cuenta cPanel | `agcapp` — activa, con AppSheet integrando MariaDB, cliente pagando |
| Buzón `soporte@agc` | Configurado en AppSheet pero **sin buzón real** en cPanel — recuperación de password rota |
| Firewall del VPS | Blanket 80/443 abierto (post-reversión Tarea 2), ipset `cloudflare-ips` conservado como andamio |
| Autorización cliente | ✅ AGC formalmente avisado — luz verde para trabajar el dominio |

**Regla operativa observada durante todo el ticket:** *disciplina estricta de read-only pre-checks antes de cualquier acción destructiva. Compartir output del terminal en cada checkpoint.*

---

## 3. Pasos ejecutados (QUÉ / COMANDO / POR QUÉ / RESULTADO)

### Paso 1 — Inventario DNS de agc-conectachile.cl 🔍

**QUÉ:** barrer todos los registros DNS del dominio desde dos perspectivas independientes.

**POR QUÉ dos vistas:** detectar drift entre lo que ve el mundo (resolver público) y lo que realmente sirve el autoritativo. Sin este contraste, un cache envenenado o un cambio no propagado pasaría desapercibido.

**BLOQUE A** — vista pública (`@1.1.1.1`) contra el dominio.
**BLOQUE B** — vista autoritativa (`@ns1.dagnstudio-vps.cl`) contra el dominio.
**BLOQUE C** — prueba de AXFR (transferencia de zona) que debe estar denegada.

**RESULTADO CRÍTICO:**
- Bloques A y B **idénticos campo a campo** → cero drift, cache limpio.
- AXFR devolvió `Transfer failed` → PowerDNS bien configurado en ese aspecto.
- Zona documentada completa (ver §7 Tabla maestra).

### Paso 2 — Buzón soporte@agc operativo end-to-end 🛠️

Sub-fases 2.1 → 2.4:

**2.1 — Pre-check de la cuenta agcapp**
- `uapi Email list_pops`, `quota -u agcapp`, `whmapi1 accountsummary`, `grep /etc/localdomains`
- Confirmó: sin buzones existentes, cuota 18% usada, límites unlimited, dominio local.

**2.2 — Crear Gmail personal para AGC**
- Cuenta: `soporte.agcconectachile@gmail.com`
- Rol dual: (a) cliente IMAP del buzón cPanel, (b) owner de la futura cuenta Cloudflare de AGC.
- Setup: saltar upsell de storage (5 GB alcanza para el uso administrativo), 2FA con teléfono personal aceptado como trade-off.

**2.3 — Crear buzón en cPanel de agcapp**
- Vía GUI: cPanel → Email Accounts → Create
- Password fuerte guardada en password manager de Oscar, cuota 1024 MB, sin welcome email.

**2.4 — Forwarder + Send mail as + tests**
- Forwarder cPanel: `soporte@agc-conectachile.cl` → `soporte.agcconectachile@gmail.com`
- Gmail "Send mail as" configurado apuntando a SMTP del VPS (`mail.agc-conectachile.cl:465` SSL).
- **Tests end-to-end confirmados con SPF/DKIM/DMARC en `pass`.** Ver §7 evidencia.

### Paso 3 — Cuenta Cloudflare a nombre de AGC 🛠️

- Signup en dash.cloudflare.com con `soporte.agcconectachile@gmail.com`.
- Verificación por email confirmada vía forwarder → llegó a Gmail.
- **2FA TOTP configurado** con app authenticator (Aegis/2FAS recomendado sobre Google Authenticator por backup cifrado).
- Backup codes guardados en password manager.
- Verificación de flujo completo (logout + login con TOTP) OK.

**Principio arquitectónico:** *el email del owner ES la propiedad de la cuenta en Cloudflare — no hay concepto separado.* Usar cuenta CF propia por cliente evita mezcla de responsabilidades y facilita handover futuro sin migración.

### Paso 4 — Replicación de la zona DNS en Cloudflare 🛠️

**4.1 — Add site** en el dashboard de CF.

**4.2 — Elección de plan:** Free. Cubre proxy DNS + Universal SSL + WAF básico + anti-bot. Suficiente para AGC.

**4.3 — Auto-import de la zona:** CF hizo scan más profundo que el barrido del Paso 1 y detectó subdominios adicionales (`autoconfig`, `cpcalendars`, `cpcontacts`, `ftp`, `webdisk`, `whm`) que cPanel autocrea. Todos fueron correctamente identificados.

**4.4 — Ajuste de proxy status por registro** (ver §7 Tabla objetivo final).

**Regla mental cristalizada:** *solo `@` y `www` van 🟠 naranja. Todo lo demás — incluidos subdominios que parecen web como `autoconfig` o `webdisk` — va ⚪ gris. Cloudflare solo proxea 80/443; cualquier puerto custom queda ciego detrás del proxy.*

**4.5 — Fix del MX auto-referenciado:**
- MX importado por CF apuntaba al apex (`agc-conectachile.cl`).
- Peligro: apex proxied resuelve a IP de CF, que no habla SMTP → correo entrante muerto.
- **Cambio ejecutado:** MX → `mail.agc-conectachile.cl` (subdominio nuevo, en gris, apuntando a `158.69.48.4`).

**4.6 — CNAME `mail` reemplazado por A record:**
- CF no permite editar directamente CNAMEs que apuntan a apex proxied.
- Solución: borrar CNAME `mail` → crear A record `mail → 158.69.48.4` en gris.

**4.7 — Validación cruzada del DKIM:**
- Clave copiada desde cPanel Email Deliverability (fuente de verdad, NO desde output de `dig`).
- Concatenación validada: fin de trozo 1 (`...EvIkiS`) + inicio de trozo 2 (`37Kphy...`) → un solo string continuo sin espacios ni comillas intermedias.
- Longitud final: 408 caracteres, consistente con RSA 2048.

**4.8 — NS asignados por CF:**
- `damon.ns.cloudflare.com`
- `emerie.ns.cloudflare.com`

### Paso 5 — Cambio de NS en NIC Chile 🛠️

**Anomalía procedimental documentada** (ver §5 Problemas): Oscar mencionó *"ya cambié los NS en NIC Chile"* cuando en realidad había **rellenado el formulario pero no hecho submit**. La validación con `dig` confirmó que el mundo aún veía PowerDNS. Se aprovechó la ventana para hacer el paso 5.A que originalmente tocaba.

**5.A — Validación pre-cambio (zona espejo en CF):**
- `dig` directo contra `damon.ns.cloudflare.com` y `emerie.ns.cloudflare.com` con la zona ya cargada pero NS aún viejos.
- **Todos los registros validados coincidieron:** SOA, NS, A, MX, TXT (SPF/DKIM/DMARC), subdominios grises.
- Ambos NS de CF respondieron idéntico → consistencia interna confirmada.

**5.B — Submit real en NIC Chile:**
- Interfaz de NIC Chile: campo "Nombre de Servidor" con `damon.ns.cloudflare.com` y `emerie.ns.cloudflare.com`.
- Checkbox "Configurar a NIC Chile como servidor secundario" → **desmarcado** (setup avanzado no aplica).
- Sin punto final (`.`) al final de los nombres — restricción de NIC Chile.
- Actualización confirmada.

### Paso 6 — Monitoreo de propagación + tests de humo ✅

**6.1 — Monitoreo de propagación:**
- 3 de 4 resolvers públicos (Cloudflare 1.1.1.1, Google 8.8.8.8, OpenDNS) ya servían `damon` + `emerie` en menos de 20 minutos.
- Quad9 (9.9.9.9) mantuvo cache viejo por más tiempo — normal, TTL respetado.
- whatsmydns.net mostró ~70% de resolvers globales migrados en ventana inicial.
- **Confirmación crítica:** `dig +short @1.1.1.1 agc-conectachile.cl A` devolvió `172.67.152.161` y `104.21.88.200` (rangos oficiales de Cloudflare) → **proxy naranja activo, IP del VPS oculta**.

**6.2 — Tests de humo (los cuatro en verde):**

| Test | Verificación | Resultado |
|---|---|---|
| A — Web + SSL | `https://agc-conectachile.cl` en incógnito + candado verde + headers `server: cloudflare` + `cf-ray` | ✅ |
| B — Correo entrante | Test desde `devops.terrible@gmail.com` a `soporte@agc-conectachile.cl` → forwarder → Gmail | ✅ |
| C — Correo saliente | Test desde Gmail con Send-mail-as → Original message con SPF/DKIM/DMARC = PASS | ✅ |
| D — Panel admin | `https://cpanel.agc-conectachile.cl:2083` → login cPanel normal (subdominio gris) | ✅ |

---

## 4. Modelo de aprendizaje — sutilezas técnicas del ticket

Sección dedicada a los conceptos clave que aparecieron y que valen documentar para el aprendizaje futuro.

### 4.1 — El MX auto-referenciado y por qué es una trampa

**Escenario original:** `MX 0 agc-conectachile.cl.` — el MX del apex apunta al propio apex.

**Analogía:** es como tener una dirección postal que dice "envía las cartas a la misma casa donde estás enviando ahora mismo". Funciona mientras la casa sea *física*. Pero si de repente la casa se convierte en un buzón virtual gestionado por otra persona (Cloudflare como proxy), las cartas no llegan al destinatario real.

**Fix universal:** crear un subdominio dedicado (`mail`) que apunte con A record a la IP real del correo, y hacer que el MX apunte al subdominio en vez del apex. Así, cuando el apex se proxea, el correo mantiene su propio camino directo al VPS.

### 4.2 — El DKIM partido en múltiples strings TXT

**Concepto:** el estándar DNS TXT (RFC 1035) impone un límite duro de **255 caracteres por string individual** dentro de un registro TXT. Cuando el contenido supera 255 chars, el servidor autoritativo lo divide automáticamente en múltiples strings al enviarlo por el wire.

**Manifestación visual en `dig`:**
```
"v=DKIM1; k=rsa; p=MIIBIj...EvIkiS" "37Kphy3NXQ6/..."
```

**Interpretación:** las comillas y el espacio entre strings son **separadores del protocolo DNS**, no contenido de la clave. Los verificadores DKIM (Gmail, Outlook, etc.) saben concatenar internamente los strings al leer el TXT.

**Riesgo en Cloudflare:** CF espera un único string continuo en su editor. Si se pega la clave con `" "` en el medio, CF puede interpretar el espacio como parte de la clave → clave inválida → `dkim=fail`.

**Fix:** copiar la clave desde la fuente de verdad (cPanel Email Deliverability), que la muestra ya concatenada. Pegar en CF como string continuo sin espacios ni comillas envolventes.

### 4.3 — Naranja vs Gris en Cloudflare: la regla de puertos

**Naranja 🟠 (Proxied):** Cloudflare intercepta el tráfico. Solo funciona para HTTP/HTTPS estándar (80/443). Aporta CDN + WAF + anti-DDoS + ocultamiento de IP.

**Gris ⚪ (DNS only):** Cloudflare solo resuelve el nombre. El tráfico va directo a la IP. Necesario para cualquier protocolo o puerto no-HTTP.

**Puertos que cPanel usa y requieren gris:**
- 2077, 2078 — WebDAV
- 2079, 2080 — CalDAV/CardDAV
- 2083 — cPanel HTTPS
- 2087 — WHM HTTPS
- 2095, 2096 — Webmail
- 25, 465, 587, 993, 995 — Correo (SMTP/IMAP/POP3)
- 21 — FTP

**Regla mnemotécnica:** *si el servicio no habla por 80 o 443, va en gris.*

### 4.4 — Los NS SPOF y por qué el orden importa

**Hallazgo raíz de la sesión previa:** los dominios `agc-conectachile.cl` y `fluxyard-appsystem.cl` delegan sus NS a `ns1/ns2.dagnstudio-vps.cl`, que a su vez es el PowerDNS del propio VPS.

**Consecuencia:** si migramos `dagnstudio-vps.cl` primero (donde vive el PowerDNS), estaríamos moviendo los NS a Cloudflare, pero los NS registrados en NIC Chile para AGC y Fluxyard siguen apuntando al `ns1.dagnstudio-vps.cl` (nombre que ahora depende de la nueva zona de CF de dagnstudio para resolver). Doble hop, riesgo alto de caída.

**Orden correcto forzado:** `AGC → Fluxyard → Dagnstudio`. Cada dominio delegado se mueve *primero* a NS propios de CF (`damon.ns.cloudflare.com`, etc.), rompiendo la dependencia con `dagnstudio-vps.cl` *antes* de que este último se migre.

---

## 5. Problemas encontrados y resolución

| # | Problema | Causa raíz | Solución |
|---|---|---|---|
| P1 | Comando `whmapi1 domain_has_dkim` inexistente en esta versión de cPanel | Mala documentación de mi parte — nombre de API no válido para EL9/AlmaLinux estándar | Retract del comando. Validación DKIM/SPF por DNS público es fuente de verdad suficiente |
| P2 | CNAME `mail → apex` no permite edición directa en CF | Limitación de UI de Cloudflare: CNAMEs que apuntan a apex proxied bloquean conversión in-place | Borrar CNAME → crear A record `mail → IP` en gris |
| P3 | Auto-import de CF trajo `mail` como CNAME 🟠 proxied y MX apuntando al apex | Import automático best-effort, no interpreta la trampa del MX auto-referenciado | Aplicar fix del §4.1 |
| P4 | Subdominios administrativos importados en 🟠 naranja (webmail, whm, cpanel, autodiscover, autoconfig, cpcalendars, cpcontacts, ftp, webdisk) | Default de CF asume que todo debería estar proxied | Toggle a gris uno por uno (9 clicks) |
| P5 | Clave DKIM importada por CF con formato `" "` en el medio | CF preserva el formato multi-string del origen | Reemplazar por clave concatenada desde cPanel (fuente de verdad) |
| P6 | Comunicación ambigua "ya cambié los NS en NIC Chile" cuando aún no se había hecho submit | Verbos "cambiar/modificar" ambiguos entre *rellenar formulario* vs *guardar cambio publicado* | Validación con `dig` reveló estado real. Nueva regla de comunicación: distinguir siempre entre *escrito* y *guardado* |
| P7 | Salto de paso: Oscar dijo "cambié NS" antes de la validación 5.A programada | Combinación de P6 + urgencia por ver resultados | Retro-adaptación: en lugar de la validación pre-cambio, se hizo validación de emergencia post-cambio. Todo salió bien, pero el patrón procedimental correcto quedó documentado como regla para próximos dominios |

---

## 6. Decisiones tomadas

| # | Decisión | Fundamento |
|---|---|---|
| **D7** | **Cuenta Cloudflare a nombre de AGC** (no compartida con DagnStudio ni cuenta personal) | Soberanía por cliente. Email owner = propiedad de la cuenta en CF. Facilita handover comercial futuro sin migración. Aprendizaje: cuando el usuario propone lo correcto, no ofrecer el atajo pragmático |
| **D8** | **Buzón cPanel real + Gmail como cliente IMAP + Send-mail-as** (patrón scm-pro replicado) | Superior a webmail (mejor UI, backup en cPanel, notificaciones push mobile, capacidad de responder con identidad del dominio). Cliente conserva webmail como alternativa |
| **D9** | **Plan Cloudflare Free para AGC** | Proxy + Universal SSL + WAF básico + anti-bot cubren todo. Plan Pro/Business no aportan valor incremental para el caso de uso |
| **D10** | **Solo `@` y `www` en 🟠 naranja; todo lo demás en ⚪ gris** | Cloudflare solo proxea 80/443. Puertos administrativos (2083, 2087, 2096, etc.) requieren acceso directo. Reduce superficie de errores futuros |
| **D11** | **MX apunta a `mail.agc-conectachile.cl` (subdominio dedicado)** en vez del apex | Fix universal del "MX auto-referenciado" — desacopla el correo del proxy naranja del apex |
| **D12** | **2FA TOTP obligatorio en cuenta CF** con backup codes en password manager | Cuenta que administra el DNS de un cliente activo sin 2FA = negligencia. TOTP > SMS. App recomendada: Aegis o 2FAS (open source, backup cifrado) |
| **D13** | **Orden de migración forzado: AGC → Fluxyard → Dagnstudio** | DNS SPOF documentado en sesión previa. No se puede tocar dagnstudio antes de haber desdelegado sus clientes |

---

## 7. Referencia técnica — tablas maestras

### 7.1 — Tabla objetivo final de la zona DNS de AGC en Cloudflare

| Tipo | Nombre | Contenido | Proxy | Rol |
|---|---|---|---|---|
| A | `@` (apex) | `158.69.48.4` | 🟠 Proxied | Web principal con protección + ocultamiento IP |
| CNAME | `www` | `agc-conectachile.cl` | 🟠 Proxied | Alias web |
| A | `mail` | `158.69.48.4` | ⚪ DNS only | Hostname SMTP/IMAP |
| A | `webmail` | `158.69.48.4` | ⚪ DNS only | Roundcube :2096 |
| A | `whm` | `158.69.48.4` | ⚪ DNS only | WHM :2087 |
| A | `cpanel` | `158.69.48.4` | ⚪ DNS only | cPanel :2083 |
| A | `autodiscover` | `158.69.48.4` | ⚪ DNS only | Outlook auto-config |
| A | `autoconfig` | `158.69.48.4` | ⚪ DNS only | Thunderbird auto-config |
| A | `cpcalendars` | `158.69.48.4` | ⚪ DNS only | CalDAV |
| A | `cpcontacts` | `158.69.48.4` | ⚪ DNS only | CardDAV |
| A | `ftp` | `158.69.48.4` | ⚪ DNS only | FTP :21 |
| A | `webdisk` | `158.69.48.4` | ⚪ DNS only | WebDAV |
| MX | `@` | `mail.agc-conectachile.cl` (prio 0) | ⚫ N/A | Enrutamiento correo entrante |
| TXT | `@` | `v=spf1 +a +mx +ip4:158.69.48.4 ~all` | N/A | SPF |
| TXT | `_dmarc` | `v=DMARC1; p=none;` | N/A | DMARC modo observador |
| TXT | `default._domainkey` | Clave DKIM 2048-bit concatenada | N/A | DKIM |
| SRV | `_autodiscover._tcp`, `_caldavs._tcp`, `_caldav._tcp`, `_carddavs._tcp`, `_carddav._tcp` | Endpoints cPanel | ⚫ N/A | Punteros de servicio |
| TXT | `_acme-challenge`, `_acme-desa...`, `_cpanel-dcv-...` | Verificaciones ACME + cPanel DCV | N/A | AutoSSL + validación de dominio |

### 7.2 — Evidencia de correo saliente con SPF/DKIM/DMARC en pass

```
Message ID: <CAJQO-RyeCg0xDqt_4jPh8xuEW7eq12YH=9i93wwfj2bufqu1mg@mail.gmail.com>
Created at: Sun, Jul 12, 2026 at 7:37 PM (Delivered after 13 seconds)
From:       Support <soporte@agc-conectachile.cl>
To:         "devops.terrible@gmail.com" <devops.terrible@gmail.com>
Subject:    Test envío como dominio AGC
SPF:        PASS with IP 158.69.48.4
DKIM:       'PASS' with domain agc-conectachile.cl
DMARC:      'PASS'
```

Este es el sello máximo de deliverability: SPF autoriza el origen, DKIM firma el mensaje, DMARC alinea ambos con el dominio del `From:`. Ningún filtro spam del mundo cuestiona el remitente.

### 7.3 — NS asignados por Cloudflare a la zona de AGC

```
damon.ns.cloudflare.com
emerie.ns.cloudflare.com
```

---

## 8. Lecciones aprendidas

### 8.1 — Metodológicas

- **Read-only pre-checks salvan vidas.** Antes de la migración pudimos catalogar cada registro, incluidos los que mi barrido inicial omitió (whm, autoconfig, cpcalendars, etc.). El auto-import de CF los detectó igual, pero saber que existen antes evita sorpresas.
- **Validar zona espejo antes del switch de NS es no-negociable.** Aunque en este ticket se saltó por confusión, el patrón queda: `dig` directo a los NS nuevos con la zona ya cargada, verificar que responden idéntico a la vieja, y solo entonces cambiar NS. Es el "cero downtime" real.
- **Los verbos operativos son ambiguos.** *"Cambié X"* / *"Modifiqué Y"* / *"Actualicé Z"* pueden significar tanto "rellené el formulario" como "guardé el cambio". Regla comunicacional: usar *"guardé"* / *"confirmé"* / *"le di submit"* cuando el cambio está publicado; usar *"rellené"* / *"escribí en el campo"* cuando aún no lo está.
- **Cuando el usuario propone lo arquitectónicamente correcto, no ofrecer el pragmático como plan B.** Fue el caso de la cuenta CF propia por cliente: mi propuesta inicial ("meterlo en tu cuenta con plan de handover") era la cómoda. Oscar corrigió, con razón, hacia el modelo correcto. La lección: cuando el patrón limpio ya es viable, ejecutarlo directamente.

### 8.2 — Técnicas específicas

- **El estándar DNS TXT parte contenidos >255 chars en múltiples strings.** Las comillas y espacios en `dig` son separadores del protocolo, no contenido. Los verificadores DKIM saben concatenar; los editores DNS como Cloudflare esperan input como string único.
- **cPanel Email Deliverability es la fuente de verdad para claves DKIM.** Mostrarla desde ahí evita errores de transcripción desde terminal → pantallazo → texto.
- **MX auto-referenciado (apex → apex) es incompatible con proxy naranja.** Siempre desacoplar con subdominio `mail` en gris.
- **CF hace mejor auto-scan del que uno espera.** Detectó `whm`, `autoconfig`, `webdisk`, SRV records y TXT de validación ACME que mi barrido manual con lista fija omitió. El auto-import como base + corrección manual de proxy status es más rápido que crear todo desde cero.

### 8.3 — Personales / de aprendiz

- **Los ✗ rojos en whatsmydns.net durante propagación NO son errores.** Son evidencia de resolvers respetando TTL cacheado del estado anterior. Peor caso mundial = 1 hora si el TTL previo era 3600s.
- **Todo cliente activo merece cuenta CF propia con 2FA.** Es la base de higiene arquitectónica multi-cliente.
- **El patrón Gmail-como-cliente-IMAP + Send-mail-as ya está validado tres veces (scm-pro × 2, vaulthost, ahora AGC).** Cuando un patrón está probado, se ejecuta directo — sin ofrecer menú de alternativas.

---

## 9. English practice — glosario de la fase

| English | Español | Uso en este ticket |
|---|---|---|
| Nameserver delegation | Delegación de nameservers | Cambiar NS en NIC Chile hacia CF |
| Zone mirroring | Replicación de zona | Copiar toda la config DNS de PowerDNS a Cloudflare |
| Proxy status | Estado del proxy | Naranja 🟠 / Gris ⚪ en CF |
| Origin IP | IP de origen | La IP real del VPS que queremos ocultar |
| Deliverability | Entregabilidad | Que el correo llegue al inbox, no al spam |
| SPF/DKIM/DMARC alignment | Alineación SPF/DKIM/DMARC | Los tres apuntando al mismo dominio del From |
| Propagation window | Ventana de propagación | Tiempo que tarda el cambio DNS en llegar a todos los resolvers |
| TTL respect | Respeto al TTL | Resolvers manteniendo cache hasta que expire el TTL declarado |
| Autoritative response | Respuesta autoritativa | La verdad del NS oficial, opuesto al cache del resolver |
| Best-effort import | Importación de mejor esfuerzo | El auto-scan de CF que trae lo que puede detectar |
| Sub-account | Sub-cuenta | Cuenta CF administrada por un tercero (no aplica aquí — usamos owner directo) |
| Handover | Traspaso | Entregar la cuenta CF al cliente formal cuando se formalice |

> *Practice:* "Nameserver delegation is a one-shot operation: once you point NS records to a new provider, the entire zone response comes from there — validate the mirror before the switch, not after."
> *Práctica:* "La delegación de nameservers es una operación de un solo tiro: una vez que apuntas los NS a un proveedor nuevo, la respuesta entera de la zona viene de ahí — valida el espejo antes del switch, no después."

---

## 10. Estado final — Definition of Done del ticket SEC-03B (parte AGC)

- [x] Inventario DNS completo con vista pública y autoritativa
- [x] Gmail `soporte.agcconectachile@gmail.com` creado con rol dual (IMAP + owner CF)
- [x] Buzón `soporte@agc-conectachile.cl` operativo en cPanel de agcapp
- [x] Forwarder cPanel → Gmail funcionando
- [x] Send-mail-as de Gmail vía SMTP del VPS con SSL:465
- [x] Cuenta Cloudflare a nombre de AGC con email verificado
- [x] 2FA TOTP configurado con backup codes en password manager
- [x] Zona DNS replicada en Cloudflare (los 20+ registros)
- [x] Fix del MX auto-referenciado aplicado (apuntando a `mail.agc-conectachile.cl`)
- [x] Nueve subdominios administrativos pasados de 🟠 a ⚪
- [x] Clave DKIM concatenada correctamente sin `" "` intermedio
- [x] Validación de zona espejo pre-cambio (contra `damon`+`emerie` con NS aún viejos)
- [x] NS cambiados en NIC Chile a `damon.ns.cloudflare.com` + `emerie.ns.cloudflare.com`
- [x] Propagación monitoreada: 3/4 resolvers públicos + whatsmydns.net global ~70%
- [x] Proxy naranja activo — IP apex resuelve a `172.67.152.161` / `104.21.88.200`
- [x] Test A — Web + Universal SSL + headers `cf-ray` ✅
- [x] Test B — Correo entrante vía forwarder ✅
- [x] Test C — Correo saliente con SPF/DKIM/DMARC = PASS ✅
- [x] Test D — Panel cPanel:2083 accesible (subdominio gris) ✅

---

## 11. Próximos pasos

### Continuación inmediata del ticket SEC-03B

**Parte 2 — Fluxyard (fluxyard-appsystem.cl):**
- Mismo playbook que AGC, sin sub-paso de buzón (proyecto en baja pendiente, no requiere correo operativo).
- Paso 1 (inventario) → Paso 3 (cuenta CF nueva) → Paso 4 (replicar zona) → Paso 5 (NS en NIC Chile) → Paso 6 (validación).
- Consideración: el buzón AppSheet en fluxyard también carece de buzón real → **evaluar** si se crea uno mínimo o se skipea al ir en baja.
- Cuenta CF: decisión pendiente si va a nombre propio (Oscar) por ser proyecto interno, o cuenta separada. Recomendación: cuenta propia porque es infra propia, no cliente.

**Parte 3 — Dagnstudio (dagnstudio-vps.cl):**
- Mismo playbook, con particularidad: preservar el buzón `admin@dagnstudio-vps.cl` que quedó como único contacto comercial.
- MX debe quedar apuntando a subdominio `mail.dagnstudio-vps.cl` en gris, apuntando al VPS.
- Este es el último dominio delegado a PowerDNS: al migrarlo, el propio PowerDNS pierde su rol autoritativo.
- **Evaluar en el futuro:** apagar PowerDNS del VPS después de esta migración (ahorro de recursos).

### Post-SEC-03B: retomar Tarea 2

Una vez los cuatro dominios (vaulthost + los tres migrados) estén en Cloudflare:
- Retomar cierre de 80/443 a IPs de Cloudflare únicamente.
- El ipset `cloudflare-ips` conservado como andamio queda listo para activar rich-rules.
- Aplicar la lección del incidente Tarea 2: verificar dominios de todas las cuentas ANTES de aplicar restricción.

### Hardening pendiente (Sprint 2 continuación)

- **SEC-12:** deshabilitar `PasswordAuthentication yes` en SSH
- **SEC-13:** restringir puertos WHM/cPanel/Webmail (2083, 2087, 2096) a VPN-only
- **Shell restriction post-deploy** de `vaulthostweb`
- **Ticket del proceso Node.js en fluxyardappsystm** corriendo como root
- **SEC-09:** fail2ban
- **SEC-05/07:** headers HSTS/CSP + WAF ModSecurity
- **Test end-to-end MercadoPago** con tarjetas de prueba oficiales

---

*Fin de la bitácora — Sprint 2 / Ticket SEC-03B parte AGC · VaultHost.cl · 12-07-2026*
