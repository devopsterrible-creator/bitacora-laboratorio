# Bitácora — Sprint 1 / Ticket 1: Descubrimiento del stack Node (solo-lectura)

> **Documento de bitácora de fase.** Registra lo ejecutado, los problemas, las decisiones y las lecciones de esta fase del proyecto VaultHost.cl.
> Complementa al runbook oficial (*VaultHost — Arquitectura y Runbook de Despliegue*); cualquier corrección al runbook detectada aquí se anota en §7.

| Campo | Valor |
|---|---|
| **Proyecto** | VaultHost VPS — sitio comercial vaulthost.cl |
| **Sprint / Fase** | Sprint 1 (Despliegue) — **Ticket 1**: descubrimiento del stack Node |
| **Tipo de trabajo** | 🔍 Solo-lectura (descubrir). No se modificó el servidor. |
| **Responsable** | Oscar (owner, dev y administrador) |
| **Acompañamiento** | Claude (ingeniero de cabecera) |
| **Fecha** | 24-05-2026 |
| **Estado** | ✅ Ticket 1 cerrado · ⏭️ Provision pendiente (Ticket 2) |
| **Método de despliegue** | Opción A — cPanel Application Manager + Phusion Passenger |

---

## Control de versiones del documento

| Versión | Fecha | Cambio | Autor |
|---|---|---|---|
| 1.0 | 24-05-2026 | Bitácora inicial del Ticket 1 (descubrimiento) | Oscar + Claude |

---

## 1. Objetivo de la fase

Verificar, **en modo solo-lectura**, qué versión de Node ofrece EasyApache 4 en el VPS y si es compatible con Next.js 16, y confirmar la disponibilidad del módulo Phusion Passenger — **todo antes de instalar o modificar nada** en el servidor.

Principio metodológico aplicado: *primero descubrir, después construir.* Cada comando de esta fase fue de lectura pura (`ls`, `dnf list/info/repoquery`, `rpm -qa`, `httpd -M`): consultan estado o metadata, no instalan ni cambian configuración.

---

## 2. Punto de partida

| Recurso | Valor |
|---|---|
| SO / Panel | AlmaLinux 9.7 (no CloudLinux) · cPanel/WHM · Apache 2.4.67 |
| Cuenta de trabajo | `vaulthostweb` (dominio vaulthost.cl) |
| Repo EA4 | `EA4-c9` (EasyApache 4 para EL9/AlmaLinux 9) |
| Estado esperado (auditoría previa) | Sin `ea-nodejs` ni Passenger instalados |

---

## 3. Requisito técnico verificado

Antes de mirar el servidor se fijó el **umbral de decisión** consultando la documentación oficial de Next.js:

- **Next.js 16 exige Node.js ≥ 20.9.0.** Node.js 18 ya **no** es soportado.
- Conclusión: buscar `ea-nodejs20` o superior. Si solo hubiera `ea-nodejs18` o menos → activar Plan B (§6.bis del runbook).

*(Fuente: documentación oficial de Next.js, versión 16.2.6 — la del proyecto.)*

---

## 4. Pasos ejecutados (QUÉ / COMANDO / POR QUÉ / RESULTADO)

### Sub-fase A — ¿Qué hay instalado hoy?

**A1 — ¿Existe algún Node de cPanel?**
- **QUÉ:** confirmar el punto de partida real, no el asumido.
- **COMANDO:** `ls -d /opt/cpanel/ea-nodejs*/ 2>/dev/null`
- **POR QUÉ:** re-verificar el estado justo antes de actuar; entre la auditoría y hoy un update pudo cambiar algo.
- **RESULTADO:** vacío → nada instalado. ✅

**A2 — ¿Passenger cargado en Apache?**
- **QUÉ:** ver si el módulo ya está activo.
- **COMANDO:** `/usr/sbin/httpd -M | grep -i passenger`
- **POR QUÉ:** si ya estuviera cargado, el ticket cambiaría (validar en vez de provisionar).
- **RESULTADO:** vacío → no cargado. ✅

### Sub-fase B — ¿Qué ofrece el repositorio? (sin instalar)

**B1 — Listar los Node disponibles para provisionar.**
- **COMANDO:** `dnf list available 'ea-nodejs*'`
- **POR QUÉ:** distinguir lo que *podemos instalar* de lo que *está instalado*.
- **RESULTADO:** disponibles `ea-nodejs10` (10.24.1), `16` (16.20.2), `18` (18.20.8), `20` (**20.20.2**), `22` (**22.22.3**). Ambos 20 y 22 superan el umbral ≥20.9. → Ver **Decisión D1**.

**B2 — Confirmar el módulo Passenger (nombre del runbook).**
- **COMANDO:** `dnf list available 'ea-apache24-mod_passenger'`
- **RESULTADO:** `Error: No matching Packages to list`. → Ver **Problema P1**.

**B3 — Confirmar versión de `ea-nodejs20`.**
- **COMANDO:** `dnf info ea-nodejs20`
- **RESULTADO:** versión **20.20.2** (≥ 20.9). ✅

### Sub-fase B-bis — Encontrar el nombre real del Passenger

**B4 — Intento con el nombre sugerido desde documentación web.**
- **COMANDO:** `dnf info ea-ruby27-mod_passenger`
- **RESULTADO:** `Error: No matching Packages to list`. → Ver **Problema P2**.

**B5 — Búsqueda *wildcard* contra el repo (la jugada correcta).**
- **QUÉ:** preguntarle al repo qué paquetes Passenger existen, sin asumir el nombre.
- **COMANDO:** `dnf list available '*passenger*'`
- **POR QUÉ:** el repo es la única fuente de verdad para nombres de paquetes.
- **RESULTADO:** el nombre real es **`ea-apache24-mod-passenger`** (¡guion medio!) **6.1.3**. También aparecen `ea-passenger-runtime`, `ea-passenger-src`, `ea-nginx-passenger` (no aplica — usamos Apache) y `-doc`.

**B6 — Companion `mod_env`.**
- **COMANDO:** `dnf list available 'ea-apache24-mod_env'`
- **RESULTADO:** **2.4.67** disponible. ✅

**B7 — ¿Algún Passenger ya instalado?**
- **COMANDO:** `rpm -qa | grep -i passenger`
- **RESULTADO:** vacío → nada instalado. ✅

### Sub-fase B-tris — Confirmación final del paquete

**B8 — Detalle del módulo correcto.**
- **COMANDO:** `dnf info ea-apache24-mod-passenger`
- **RESULTADO:** **6.1.3**, "Phusion Passenger application server". La descripción confirma soporte para **Ruby, Python, Node.js y Meteor** → se disipa la duda del prefijo "ruby". ✅

**B9 — Árbol de dependencias.**
- **QUÉ:** confirmar que el módulo arrastre el runtime.
- **COMANDO:** `dnf repoquery --requires --resolve ea-apache24-mod-passenger`
- **POR QUÉ:** entender qué se instala junto al módulo *antes* de provisionar.
- **RESULTADO:** confirma **`ea-passenger-runtime` (1.0)** como dependencia → marcando solo el módulo, EasyApache jala el runtime. Aparece además un **`nodejs 20.20.2` (NodeSource) + `nsolid`** → ver **Problema P3**.

---

## 5. Problemas encontrados y cómo se resolvieron

### P1 — Nombre de paquete equivocado en el runbook
- **Síntoma:** `dnf list available 'ea-apache24-mod_passenger'` → "No matching Packages".
- **Causa:** el runbook (§6 paso 3) traía el nombre con **guion bajo** (`mod_passenger`), que no existe.
- **Solución:** búsqueda *wildcard* `'*passenger*'`; el nombre real usa **guion medio** (`ea-apache24-mod-passenger`).
- **Impacto:** corrección al runbook (§7).

### P2 — El nombre sugerido desde docs web tampoco existía
- **Síntoma:** `dnf info ea-ruby27-mod_passenger` → "No matching Packages".
- **Causa:** la documentación web mayoritariamente describe el **Node.js Selector de CloudLinux** o generaciones antiguas de EA4. Este VPS es **AlmaLinux estándar** con repo **`EA4-c9`**, que **modernizó el empaquetado**: Passenger quedó partido en *runtime* (`ea-passenger-runtime`) + módulo por servidor web (`ea-apache24-mod-passenger`).
- **Solución:** confiar en el resultado del repo, no en el nombre de ninguna doc.
- **Lección directa:** *ni la documentación ni el asistente son la fuente de verdad; el repo sí.* (Lección reforzada porque el nombre sugerido por el asistente también falló — se verificó contra el repo y se corrigió.)

### P3 — El "Node fantasma" de Passenger
- **Síntoma:** el `repoquery` lista `nodejs-2:20.20.2-1nodesource` + `nsolid-0:20.20.2` como dependencias, **separados** del `ea-nodejs22` que elegimos para la app.
- **Causa:** Passenger declara una dependencia de un `nodejs` genérico; los paquetes `ea-nodejs` viven en `/opt/cpanel/` y **no** proveen esa capacidad, así que el resolver la satisface con el `nodejs` de sistema (NodeSource 20.x).
- **Riesgo:** ambigüedad sobre **qué Node ejecuta realmente la app** (¿el `ea-nodejs22` deseado o el `nodejs 20` de Passenger?).
- **Estado:** **NO es bloqueador.** Ambos Node superan ≥20.9, así que Next.js 16 corre igual. La diferencia es quedar en 20 (cerca de EOL) vs 22 (con runway).
- **Honestidad técnica:** la documentación consultada **no resolvió limpio** este punto para AlmaLinux estándar (casi todo es CloudLinux). Por eso **no se asume**: se convierte en checkpoint verificable.
- **Acción (Ticket 2):**
  1. Al registrar la app en Application Manager, confirmar que se pueda **apuntar/elegir** `ea-nodejs22`.
  2. Post-deploy, **medir en caliente** la versión real (`process.version` / `node -v` dentro del entorno de la app). *La verdad del runtime se mide, no se supone.*

---

## 6. Decisiones tomadas

| # | Decisión | Por qué | Nota |
|---|---|---|---|
| **D1** | Usar **`ea-nodejs22`** (22.22.3) en vez de `ea-nodejs20` | Node 20 llega a *end-of-life* ~abril 2026 (prácticamente ahora); Node 22 es LTS con soporte ~hasta abril 2027. Next 16 corre en ambos. Arrancar producción sobre un runtime en EOL = deuda técnica día uno. | Candidata a **ADR-06** en el runbook |
| **D2** | Nombre correcto del módulo = **`ea-apache24-mod-passenger`** (guion medio) | Es el único que existe en el repo `EA4-c9`. | Corrige runbook §6/§4 |
| **D3** | Confirmar camino **estándar** (no CloudLinux): Application Manager + Passenger | El VPS es AlmaLinux, no CloudLinux → **no** se usa Node.js Selector ni `alt_mod_passenger`. Calza con la Opción A ya adoptada (ADR-02). | Sin cambios de arquitectura |

---

## 7. Correcciones al runbook (a aplicar en su v1.1)

- **§4 y §6 paso 3:** `ea-apache24-mod_passenger` (guion bajo, inexistente) → **`ea-apache24-mod-passenger`** (guion medio) + `ea-apache24-mod_env`.
- **§4 y §6 paso 3:** `ea-nodejs20` → **`ea-nodejs22`**.
- **Nota nueva (§6):** documentar el "Node fantasma" de Passenger (NodeSource 20.x) y el **checkpoint de binding de runtime** a realizar en el Provision.
- **Registro de decisiones (§5):** agregar **ADR-06** (elección de `ea-nodejs22` por EOL de Node 20).

---

## 8. Lista de compras para el Ticket 2 (Provision) 🛠️

Paquetes a marcar en EasyApache → *Customize*:

1. **`ea-nodejs22`** (22.22.3) — runtime de la app.
2. **`ea-apache24-mod-passenger`** (6.1.3) — módulo Apache; arrastra `ea-passenger-runtime`.
3. **`ea-apache24-mod_env`** (2.4.67) — companion.

> En la GUI, el toggle de Passenger se encuentra bajo la sección llamada *"Ruby via Passenger"* (nombre confuso, pero correcto). Marcar y **no** dar Provision hasta revisar.

---

## 9. Lecciones aprendidas

- **El repositorio es la *single source of truth* para nombres de paquetes.** La documentación se desactualiza; `dnf` no miente.
- **La precisión del *string* importa:** un guion bajo vs. un guion medio fue la diferencia entre "encontrado" y "No matching Packages".
- **El solo-lectura paga:** cazamos dos nombres equivocados y un Node fantasma **sin tocar** el servidor.
- **Tres preguntas distintas, tres comandos distintos:** *¿qué está disponible?* (`dnf list available`) · *¿qué está instalado?* (`rpm -qa`) · *¿qué ejecuta realmente?* (verificación en caliente). No confundirlas.
- **Cuando la realidad contradice la doc, se corrige la doc** — no se ignora el hallazgo.
- **Honestidad técnica:** si la documentación no resuelve un punto, se marca como "verificar empíricamente"; no se rellena con suposiciones.

---

## 10. English practice — glosario de la fase

| English | Español | Uso en esta fase |
|---|---|---|
| Read-only discovery | Descubrimiento solo-lectura | La fase completa: consultar sin modificar |
| Single source of truth | Única fuente de verdad | El repo manda sobre la doc |
| Package / capability *provides* | Capacidad que provee un paquete | Por qué Passenger jaló el `nodejs` de sistema |
| Transitive dependency | Dependencia transitiva | El "Node fantasma" en el árbol |
| Runtime | Entorno de ejecución | Qué Node ejecuta la app |
| End-of-life (EOL) | Fin de soporte | Razón para preferir Node 22 |
| Empirically | Empíricamente | Medir la versión en caliente, no asumirla |

> *Practice:* "Don't trust the package name in the docs — query the repository, it's the single source of truth. And verify the runtime empirically."
> *Práctica:* "No confíes en el nombre del paquete en la doc — consulta el repositorio, que es la única fuente de verdad. Y verifica el runtime empíricamente."

---

## 11. Estado final — Definition of Done del Ticket 1

- [x] Confirmado que no había Node ni Passenger instalados.
- [x] `ea-nodejs22` (22.22.3) disponible → cumple Next.js 16, con runway hasta ~2027.
- [x] `ea-apache24-mod-passenger` (6.1.3) disponible → arrastra `ea-passenger-runtime`.
- [x] `ea-apache24-mod_env` (2.4.67) disponible.
- [x] Correcciones al runbook identificadas (§7).
- [ ] ⚠️ **Pendiente Ticket 2:** verificar binding del runtime a `ea-nodejs22` (Node fantasma).

---

## 12. Próximos pasos

**Ticket 2 — Provision (primer paso 🛠️ real):**
- Marcar los 3 paquetes (§8) en EasyApache → *Review* → *Provision*.
- Validar: `/usr/sbin/httpd -M | grep -i passenger` debe listar el módulo.
- **Checkpoint:** confirmar/forzar binding a `ea-nodejs22` y medir `node -v` en caliente.

**Tickets posteriores del Sprint 1 (aún no iniciados):**
- Habilitar Application Manager y registrar la app (`server.js` standalone).
- AutoSSL + redirección HTTP→HTTPS.
- MercadoPago a **producción** (webhook y back_urls a vaulthost.cl).
- Reemplazar WhatsApp placeholder `56900000000` → **`56971456955`**.
- Pruebas end-to-end (checkout + voucher + webhook).

---

*Fin de la bitácora — Sprint 1 / Ticket 1 · VaultHost.cl*
