# Reflexión y evolución del diseño — Fase 3B parte 1

> **Documento hermano** de `fase-05-ejecucion-wireguard-y-tarea2.md`.
> Mientras la bitácora de ejecución responde *"¿qué se hizo?"*, este documento responde *"¿qué aprendimos y cómo debe evolucionar el diseño?"*. Envejece mejor que la bitácora técnica y suele ser el que se relee.

| Campo | Valor |
|---|---|
| **Proyecto** | VaultHost VPS — hardening de infraestructura |
| **Sesión de referencia** | Sprint 2 / Fase 3B, sesión 1 (09-07-2026) |
| **Autor** | Oscar + Claude |

---

## 1. Resumen de la sesión en una frase

Intentamos cerrar el bypass de origen del VPS confiando en que todos los dominios estaban detrás de Cloudflare; descubrimos que no era así, revertimos limpio, y salimos con **causa raíz identificada, andamio dejado, y un mejor mapa de la infraestructura del que teníamos al empezar**.

---

## 2. La hipótesis con la que entramos y la realidad que encontramos

### Hipótesis inicial (implícita, no cuestionada)

> "Con Cloudflare ya proxeando `vaulthost.cl`, restringir 80/443 a rangos IP de Cloudflare cierra el bypass sin efectos colaterales, porque todo el tráfico legítimo al VPS ya viene por Cloudflare."

### Lo que la realidad reveló

El servidor aloja **5 vhosts**, no 1. Solo **1 de los 5 dominios** (`vaulthost.cl`) estaba efectivamente registrado y proxeado en la cuenta Cloudflare de VaultHost (confirmado por dashboard: *1 - 1 of 1 items*). Los otros tres vhosts que **sí viven en el VPS** — `dagnstudio-vps.cl`, `agc-conectachile.cl`, `fluxyard-appsystem.cl` — resuelven directo a `158.69.48.4` en el DNS público. Cerrar 443 a "solo Cloudflare" los dejó fuera de la lista blanca.

Aparte, `scm-pro.cl` — que en el `dig` inicial parecía proxeado — vive en otro origen (SMART 10 reseller). El dato del DNS era correcto pero engañoso: **estar proxeado por Cloudflare no significa apuntar a este VPS**.

### Por qué la hipótesis pareció razonable

- El registro histórico del proyecto describía Cloudflare como "activo" (era cierto — para vaulthost.cl).
- La verificación por `whatsmydns.net` había mostrado IPs de Cloudflare (era cierto — para vaulthost.cl).
- Nadie había mapeado explícitamente **por vhost**, dominio a dominio, cuáles estaban proxeados y cuáles no.

---

## 3. Los conceptos técnicos que quedaron confirmados con evidencia

Estos son los aprendizajes duros de la sesión, sacados de mediciones reales, no de teoría de libro.

### 3.1 Un firewall opera en capa L3/L4, no distingue vhosts

`firewalld` filtra por IP de origen y puerto. **No sabe qué dominio traía la petición HTTP.** Un candado en el puerto 443 se aplica igual a todos los `VirtualHost *:443` de Apache. En un servidor multi-tenant, cualquier cambio de red afecta a **todas** las cuentas cPanel del servidor, no solo a la que se está pensando.

*Práctica en inglés:* *"A firewall rule is host-agnostic — it drops packets based on source IP and port, oblivious to which virtual host inside Apache would have received them."*

### 3.2 Cloudflare protege HTTP/HTTPS, no bases de datos

En plan Free, Cloudflare proxea 80/443 (y algunos puertos alternativos). **No proxea 3306 (MySQL/MariaDB).** La conexión de Google AppSheet → MariaDB del VPS **sigue yendo directo**, protegida por el ipset `appsheet-ips` que ya existe. Meter un dominio a Cloudflare mejora la protección del sitio web, pero **no cambia nada en la ruta de la base de datos**. Son dos planos independientes.

### 3.3 Un dominio "proxeado por Cloudflare" puede apuntar a cualquier origen

El proxy naranja de Cloudflare simplemente reenvía a la IP que tú configures como origen. Que el `dig` público muestre un rango Cloudflare no dice nada sobre qué servidor está detrás. Para saber qué origen tiene un dominio en Cloudflare, la fuente de verdad es el **dashboard**, no el DNS público.

### 3.4 En un shell zsh, la flecha unicode `→` se interpreta como comando

Un detalle chico pero real: pegando texto con `→` en un heredoc, zsh reportó *"no se encontró la orden"* diez veces. Reemplazar por `->` ASCII resolvió. Regla: en scripts para pegar en terminal, **solo ASCII**.

### 3.5 El estado "active (exited)" de un servicio systemd puede ser normal

`wg-quick@wg0.service` mostró `Active: active (exited)`. Parece error, no lo es: el script `wg-quick` es de tipo *one-shot* — configura la interfaz en el kernel y termina exitosamente. La interfaz sigue viva, aunque el proceso del script haya salido. Lección: no interpretar el estado de un `.service` como si fuera un daemon convencional sin leer su tipo primero.

---

## 4. Errores metodológicos de esta sesión (y su corrección)

Documentarlos honestamente vale más que esconderlos.

### 4.1 Mapa de dependencias tácito, no explícito

**El error:** entré a la Tarea 2 con un mapa mental de "vaulthost = todo el server", sin haber inventariado antes qué otros dominios comparten Apache y en qué estado están respecto a Cloudflare.

**El fix metodológico:** *antes de tocar cualquier puerto en un servidor multi-tenant, mapear qué dominios/cuentas dependen de ese puerto y en qué estado están respecto al filtro propuesto*. Esto se convierte en un pre-check obligatorio en el runbook.

### 4.2 Reconocer al `dig` como pista, no como verdad

**El error:** interpreté "resuelve a IP de Cloudflare" como "vive en nuestro Cloudflare, apuntando a este VPS". Ambos son falsos independientemente.

**El fix metodológico:** cuando la fuente de verdad es un dashboard (Cloudflare, cPanel, WHM), **consultar el dashboard**, no intentar inferirlo desde afuera con herramientas de red.

### 4.3 Confiar en la insistencia del usuario cuando algo no cuadra

**El error:** en tres momentos distintos de la sesión, Oscar dijo "espera, algo no cuadra" (reinicio de Apache, scm-pro fuera del VPS, dashboard de Cloudflare con solo vaulthost). En los tres tenía razón; en los tres tuve que rectificar la línea de investigación.

**El fix metodológico:** la insistencia del que está viendo el sistema real **es señal, no ruido**. Cuando el operador dice "no cuadra", frenar y remapear, no seguir empujando el diagnóstico anterior.

### 4.4 Reversión rápida es un logro, no una derrota

**Reconocimiento honesto:** el ciclo total entre detección del impacto → reversión → confirmación de recuperación (AGC 200 OK) fue del orden de 15 minutos. En operación real, esa métrica (MTTR — Mean Time To Recovery) importa más que "cero incidentes", porque cero incidentes no existe. Un equipo maduro se mide por su capacidad de recuperación, no por su capacidad de evitar toda falla.

---

## 5. Lo que ganamos aunque el ticket no se cerró

Lista honesta:

1. **Túnel WireGuard verificado end-to-end**, split-tunnel confirmado en `/32` (la configuración más estricta posible). Miedo original de Oscar sobre VPNs → desmontado con evidencia.
2. **Mapa real de la infraestructura**, dominio a dominio, con el dashboard de Cloudflare como fuente de verdad.
3. **Andamio de firewall dejado listo** — ipset `cloudflare-ips` con las 15 IPs vigentes + rich-rules. Cuando se cierre el candado en la siguiente ronda, es una sola operación de dos comandos.
4. **Puerto 3000 anotado con contexto completo**: proceso vivo, corriendo como root, en cuenta supuestamente congelada. Contradicción documentada, ticket abierto.
5. **Método de reproducción segura de fallas confirmado** con Kali en NAT + curl — mismo método servirá para SEC-12 y SEC-13.

---

## 6. Nuevos desafíos que emergieron

| Ticket nuevo | Origen | Prioridad |
|---|---|---|
| **SEC-03B** — Migrar dominios activos a Cloudflare | Bloquea cierre de Tarea 2 | Alta |
| **SEC-03C** — Decidir destino de `fluxyard-appsystem.cl` | Cuenta "congelada" con proceso vivo — inconsistente | Media |
| **Node-root** — Investigar `pid=1450` como root en cuenta congelada | Violación de menor privilegio detectada | Media-alta |

---

## 7. Evolución del diseño de la Fase 3B (v1 → v2)

**Fase 3B v1** (planificación original de esta mañana):
1. Candar 80/443 a Cloudflare
2. Cerrar paneles admin a VPN
3. SSH key-only
4. cPHulk + fail2ban
5. 2FA WHM

**Fase 3B v2** (post-sesión):
0. **SEC-03B — Migrar dominios activos a Cloudflare** *(nuevo prerequisito)*
1. Candar 80/443 a Cloudflare *(desbloqueado por 0)*
2. Cerrar paneles admin a VPN *(independiente, pre-validado — se puede hacer aparte)*
3. SSH key-only *(independiente — puede hacerse en paralelo con 2)*
4. cPHulk + fail2ban
5. 2FA WHM

**Cambio de diseño clave:** los puntos 2 y 3 (paneles y SSH) **no dependen** del 1 (candado Cloudflare). Se pueden ejecutar en las próximas sesiones sin esperar a que se resuelva SEC-03B. Esto **desbloquea progreso paralelo**.

---

## 8. Aprendizajes de trabajo (metodológicos y personales)

### 8.1 De ingeniería

- **Constructivo antes que destructivo** — se cumplió (ipset + rich-rules antes de remover servicios).
- **Read-only antes de modificar** — se cumplió (WG-00 completo antes de la Tarea 2).
- **Auditar output de terminal en cada checkpoint** — se cumplió (el `firewall-cmd --list-all` intermedio permitió confirmar la rich-rule antes de destructivo).
- **Impacto multi-tenant como pre-check obligatorio** — **nuevo, incorporar al runbook**.

### 8.2 De colaboración humano-IA

- Cuando el operador humano insiste, **el humano probablemente tiene razón**. El asistente tiene que remapear su modelo, no defenderlo.
- La verificación cruzada con el dashboard (fuente de verdad interna) es más confiable que cualquier inferencia externa por red.
- La bitácora doble — ejecución + reflexión — evita el antipatrón de mezclar "qué comando corrí" con "qué aprendí" en un solo documento que no se relee.

### 8.3 Personales (Oscar)

- Poder analítico confirmado en vivo: tres frenos, tres aciertos.
- La necesidad de "ver el mapa completo antes de tocar una pieza" es un rasgo *thinking-in-systems*, propio de ingenieros senior. No es defecto, es fortaleza.
- Falta de retención de conceptos — abordada con el ticket personal *"segundo cerebro"* que se abre a continuación.

---

## 9. Ideas para el "pensadero" — sistema de retención de conocimiento

Salieron de la conversación al cierre. Se implementan en próximas sesiones:

1. **Índice maestro** encima de las bitácoras: un solo `.md` tipo "mapa del pensadero" que apunte *"si el problema es X, ver fase Y sección Z"*. Convierte el conjunto de bitácoras de "archivo" a "base de conocimiento consultable".
2. **Cierre estilo Feynman** en cada fase: una frase final en lenguaje llano — "hoy aprendí que X". Refuerza retención por reformulación.
3. **Repaso 24h/7d/30d**: releer bitácora 5 minutos al día siguiente, a la semana, al mes. *Spaced repetition* aplicado a documentación técnica.
4. **Diagrama Mermaid de cierre** en cada bitácora: "antes → después" mínimo. Aprovecha el sesgo visual del autor para retención de largo plazo.

---

## 10. Postmortem breve (formato empresarial estándar)

Para practicar cómo se reporta este tipo de sesión en contexto profesional real:

| Campo | Contenido |
|---|---|
| **Incident** | 3 dominios (dagnstudio-vps.cl, agc-conectachile.cl, fluxyard-appsystem.cl) inaccesibles desde internet por ~15 minutos tras aplicar restricción de firewall en 80/443. |
| **Root cause** | Restricción aplicada asumiendo que todos los vhosts del servidor estaban proxeados por Cloudflare; en realidad solo 1 de 4 lo estaba. Los otros 3 resuelven directo a la IP del VPS y fueron descartados por el filtro. |
| **Impact** | Cliente activo (AGC) afectado. Sin pérdida de datos, sin exposición de secretos, sin degradación permanente. |
| **Detection** | Operador humano detectó impacto visualmente y escaló al ingeniero de guardia. |
| **Recovery** | Reversión del cambio de firewall (2 comandos). Validación con `curl` externo. Total: ~15 minutos desde detección hasta recuperación confirmada. |
| **Action items** | SEC-03B (migrar dominios a Cloudflare), pre-check de mapeo multi-tenant obligatorio en runbook, revisión de proceso root en cuenta congelada. |
| **What went well** | Ipset creado en paralelo antes de destructivo permitió reversión limpia. Cliente activo recuperado sin daño. Causa raíz identificada por operador humano en tiempo real. |
| **What to improve** | Pre-mapeo de dependencias por vhost antes de tocar firewall de red. Consulta al dashboard como fuente de verdad antes de inferir estado desde afuera. |

---

## 11. English practice — glosario acumulado de la sesión

| English | Español | Uso en esta sesión |
|---|---|---|
| Origin bypass | Bypass de origen | Riesgo cuando la IP real es alcanzable sin pasar por CDN |
| Multi-tenant server | Servidor multi-inquilino | El VPS con 5 cuentas cPanel compartiendo Apache |
| Blast radius | Radio de impacto | Cuántos dominios afectó el cambio (3 de 4 vhosts locales) |
| MTTR — Mean Time To Recovery | Tiempo medio a la recuperación | ~15 min en este incidente |
| Postmortem | Análisis post-incidente | La sección 10 de este documento |
| Root cause analysis (RCA) | Análisis de causa raíz | El proceso completo de esta sesión |
| Rollback | Reversión | La operación de re-agregar `http`/`https` |
| Scaffolding | Andamiaje | El ipset + rich-rules que dejamos listas para la próxima ronda |
| Source of truth | Fuente de verdad | El dashboard de Cloudflare vs. el `dig` público |
| Split-tunnel | Túnel dividido | La configuración de WireGuard con `AllowedIPs=10.8.0.1/32` |

> *Practice:* *"A production incident is not defined by whether something broke, but by how fast and cleanly you recovered — that's what MTTR measures, and it's the real signal of engineering maturity."*
>
> *Práctica:* *"Un incidente en producción no se define por si algo se rompió, sino por qué tan rápido y limpiamente te recuperaste — eso es lo que mide MTTR, y es la señal real de madurez de ingeniería."*

---

## 12. Frase de cierre estilo Feynman (para retención)

> **Un firewall no distingue dominios; solo ve IPs y puertos. Por eso, en un servidor con varios sitios adentro, no puedes cerrar un puerto sin haber revisado primero, sitio por sitio, quién depende de él y por dónde entra.**

Esa frase, si mañana no me acuerdo de nada más, resume el 80% de esta sesión.

---

*Fin del documento de reflexión — Fase 3B parte 1 · VaultHost.cl*
