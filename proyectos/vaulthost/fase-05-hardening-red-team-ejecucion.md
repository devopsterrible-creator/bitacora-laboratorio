# Bitácora — Sprint 2 / Hardening Red Team: WG-00 + Tarea 2 (ejecución)

> **Documento de bitácora técnica de fase.** Registra lo ejecutado, los outputs reales, los problemas encontrados y las decisiones tomadas durante la sesión.
> Complementa al runbook oficial (*VaultHost — Arquitectura y Runbook de Despliegue*).
> **Cierre en una frase (regla del pensadero):** *Un firewall opera en L3/L4 y no distingue vhosts; cerrar un puerto en un servidor multi-tenant lo cierra para todos los dominios que compartan ese puerto.*

| Campo | Valor |
|---|---|
| **Proyecto** | VaultHost VPS — sitio comercial vaulthost.cl |
| **Sprint / Fase** | Sprint 2 (Hardening) — **Tickets WG-00 + Tarea 2** |
| **Tipo de trabajo** | 🔍 Verificación + 🛠️ Modificador + 🔄 Reversión controlada |
| **Responsable** | Oscar (owner, dev y administrador) |
| **Acompañamiento** | Claude (ingeniero de cabecera) |
| **Fecha** | 09-07-2026 (sesión nocturna) |
| **Estado** | ✅ WG-00 cerrado · ⚠️ Tarea 2 intentada + revertida + causa raíz identificada |
| **Método de trabajo** | Red-only-before-modify + rich-rules + ipset (patrón heredado de SEC-02) |

---

## Control de versiones del documento

| Versión | Fecha | Cambio | Autor |
|---|---|---|---|
| 1.0 | 09-07-2026 | Bitácora inicial de la sesión (WG-00 cerrado + Tarea 2 revertida) | Oscar + Claude |

---

## 1. Objetivos originales de la fase

Ejecutar los primeros dos frentes del hardening Red Team → Blue Team:

- **WG-00:** verificar que WireGuard (VPN administrativa) esté correctamente instalado y configurado en ambos extremos (servidor VPS + cliente Windows 11), sin dar por buenos supuestos históricos.
- **Tarea 2 (Cloudflare origin lock):** restringir 80/443 en firewalld para que solo acepte conexiones desde IPs oficiales de Cloudflare, cerrando el bypass de origen que permitiría a un atacante saltarse el WAF pegando directo a la IP real del VPS.

Ambos tickets forman parte de la Fase 3B del hardening — la "puerta de administración segura" (WG-00) y el "candado del origen web" (Tarea 2).

---

## 2. Ticket WG-00 — Verificación WireGuard (🔍 solo lectura)

### 2.1 Contexto y por qué se abrió el ticket

Aunque el registro histórico del proyecto mencionaba la subred `10.8.0.0/24` en las rich-rules de firewalld, no había evidencia concreta en el historial de conversaciones de una instalación acompañada de WireGuard (ni servidor ni cliente). El proceso pudo haber ocurrido con otro asistente. Antes de basar SEC-12/SEC-13 sobre la VPN, se decide verificar el estado real con comandos read-only en ambas puntas.

### 2.2 Ejecución (lado servidor VPS)

**A1 — Interfaz WireGuard activa:**
```bash
sudo wg show
```
Resultado:
```
interface: wg0
  public key: 9snolisG9DxAA4hcdyqBPgEECBuSKBexkCuhpGjO3VY=
  private key: (hidden)
  listening port: 51820

peer: SueSMiclgrn9LxZOaPnM/TTfb3PYl0gNWfuU05o5Wis=
  endpoint: 190.21.129.60:61764
  allowed ips: 10.8.0.2/32
```
✅ Interfaz `wg0` activa, un peer registrado (`allowed ips: 10.8.0.2/32` → apunta al Windows 11 personal).

**A2 — IP interna del túnel en el servidor:**
```bash
ip a show wg0
```
Resultado: `inet 10.8.0.1/24 scope global wg0`, estado `UP,LOWER_UP` ✅

**A3 — Archivo de configuración + servicio systemd:**
```bash
sudo ls -la /etc/wireguard/
systemctl status wg-quick@wg0
```
Resultado:
- `server_private.key`, `server_public.key`, `wg0.conf` presentes, permisos `600` (`rw-------`) ✅
- Servicio `enabled` + `active (exited)` (estado esperado para `wg-quick@`; el script arranca la interfaz y termina; la interfaz queda viva en el kernel)
- Warning menor en log: `RTNETLINK answers: Network is unreachable` durante el arranque, en subproceso separado. No afecta funcionamiento (verificable con `ip a show wg0`).

**A4 — Consistencia con reglas de firewall existentes (SEC-02):**
```bash
sudo firewall-cmd --list-all
```
Confirmadas rich-rules para 22, 2087, 3306 desde `10.8.0.0/24`. También detectado el mismo antipatrón conocido: `ssh` sigue en la lista de services de la zona pública, y 2083/2087/2096 en `ports`, por lo que hoy **la rich-rule de VPN aún no restringe nada** (contenido pendiente de SEC-12/SEC-13, no de WG-00).

### 2.3 Ejecución (lado cliente Windows 11)

Del panel de la app WireGuard:

| Campo | Valor | Lectura |
|---|---|---|
| Direcciones | `10.8.0.2/24` | Coincide con lo registrado en el servidor |
| **AllowedIPs** | **`10.8.0.1/32`** | ✅ **Split-tunnel estricto**: solo tráfico a `10.8.0.1` cruza el túnel. Netflix, navegación, todo lo demás sigue por la conexión normal. |
| Endpoint | `158.69.48.4:51820` | Correcto |
| Persistent keepalive | `25` | Bueno para conexiones residenciales detrás de NAT |

### 2.4 Prueba de activación end-to-end

Con el túnel activado en Windows:

**Servidor:**
```
peer: SueSMiclgrn9LxZOaPnM/TTfb3PYl0gNWfuU05o5Wis=
  endpoint: 190.21.142.89:64905
  allowed ips: 10.8.0.2/32
  latest handshake: 1 minute, 50 seconds ago
  transfer: 308 B received, 92 B sent
```

**Windows:**
- Estado: Activo ✅
- Último saludo: hace 25 segundos
- Transferir: 184 B received, 552 B sent

**Prueba adicional (navegador):** con la VPN activa, `https://10.8.0.1:2087` cargó WHM correctamente (advertencia esperada de certificado por hostname vs IP; conexión igualmente cifrada por HTTPS).

**Detalle técnico anotado:** el `ping 10.8.0.1` ejecutado en la sesión SSH del servidor no cruza el túnel (es loopback del propio VPS a sí mismo, ~0.1ms). La prueba del handshake criptográfico + contadores de transferencia en ambos lados independientes son la evidencia real del túnel funcionando.

### 2.5 Definition of Done WG-00 ✅

- [x] Paquete WireGuard instalado y interfaz `wg0` activa
- [x] Peer emparejado correctamente entre servidor (`10.8.0.1`) y Windows (`10.8.0.2`)
- [x] Split-tunnel confirmado (`AllowedIPs = 10.8.0.1/32`) — resuelve el miedo histórico del operador respecto a las VPN "que estorban todo"
- [x] Servicio `wg-quick@wg0` `enabled` (persiste tras reinicio)
- [x] Handshake y transferencia bidireccional confirmados
- [x] Acceso WHM via `https://10.8.0.1:2087` validado
- [x] Ticket queda como pre-requisito **cumplido** para SEC-12 y SEC-13

---

## 3. Ticket Tarea 2 — Candar 80/443 a IPs de Cloudflare (🛠️ intentado + revertido)

### 3.1 Diseño del ticket

**Objetivo:** replicar el patrón exitoso de SEC-02 (ipset + rich-rule + remover apertura blanket) aplicado a 80/443, cerrando el bypass de origen que permite conectar directo a la IP real del VPS saltándose Cloudflare.

**Fuente de verdad para IPs de Cloudflare:** `cloudflare.com/ips` — 15 rangos IPv4 oficiales al día de la sesión.

**Regla operativa (heredada de SEC-02):** construir el permiso nuevo antes de remover el permiso blanket; verificar en cada paso; nunca al revés.

### 3.2 Ejecución Fase 1 — Construcción

**Creación del ipset y carga de las 15 IPs oficiales:**
```bash
firewall-cmd --permanent --new-ipset=cloudflare-ips --type=hash:net

for ip in \
  103.21.244.0/22 103.22.200.0/22 103.31.4.0/22 \
  104.16.0.0/13 104.24.0.0/14 108.162.192.0/18 \
  131.0.72.0/22 141.101.64.0/18 162.158.0.0/15 \
  172.64.0.0/13 173.245.48.0/20 188.114.96.0/20 \
  190.93.240.0/20 197.234.240.0/22 198.41.128.0/17; do
    firewall-cmd --permanent --ipset=cloudflare-ips --add-entry=$ip
done

firewall-cmd --reload
```
✅ 15 entradas cargadas, verificadas con `--get-entries`.

**Agregado de rich-rules (permitir 80/443 desde el ipset):**
```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source ipset="cloudflare-ips" port port="80" protocol="tcp" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source ipset="cloudflare-ips" port port="443" protocol="tcp" accept'
firewall-cmd --reload
```
✅ Ambos `success`. Zona pública ahora tenía **doble permiso** para 80/443 (rich-rule específica de Cloudflare + services `http`/`https` blanket). Sitio operando normalmente.

### 3.3 Ejecución Fase 2 — Candado (destructivo)

**Remoción de los services blanket:**
```bash
firewall-cmd --permanent --remove-service=http
firewall-cmd --permanent --remove-service=https
firewall-cmd --reload
```
✅ Confirmado por listado posterior: `services` ya sin `http https`, `rich rules` con las dos entradas de `cloudflare-ips` activas.

### 3.4 Prueba desde perspectiva externa (Kali en NAT)

**Bypass directo por IP real:**
```bash
curl -k -m 5 https://158.69.48.4
# curl: (28) Connection timed out after 5002 milliseconds
```
✅ Timeout → bypass efectivamente cerrado.

**Camino legítimo por dominio:**
```bash
curl -k -m 5 https://vaulthost.cl
# HTML completo devuelto, sitio operando
```
✅ vaulthost.cl sigue vivo (resuelve `104.21.7.38` → Cloudflare → VPS acepta porque origen es rango CF).

### 3.5 Observación del operador que cambió todo

Operador reporta: **"dagnstudio-vps.cl se cayó"**. Ese solo dato fuerza revisión inmediata del impacto multi-tenant.

Verificación en batch (Kali, con `-> ` en lugar de flecha unicode):
```bash
for d in vaulthost.cl scm-pro.cl agc-conectachile.cl dagnstudio-vps.cl fluxyard-appsystem.cl; do
  echo -n "$d -> HTTP "
  curl -k -m 5 -o /dev/null -w "%{http_code} (resuelve: %{remote_ip})\n" https://$d
done
```

**Resultado (con candado cerrado, momento del incidente):**

| Dominio | Resuelve a | Estado durante candado |
|---|---|---|
| vaulthost.cl | `104.21.7.38` (Cloudflare) | 🟢 Vivo |
| scm-pro.cl | `104.21.36.34` (Cloudflare) | ⚠️ Proxeado, pero apunta a otro origen (SMART 10, no este VPS) |
| agc-conectachile.cl | `158.69.48.4` (VPS directo) | 🔴 **CAÍDO — cliente activo** |
| dagnstudio-vps.cl | `158.69.48.4` (VPS directo) | 🔴 Caído |
| fluxyard-appsystem.cl | `158.69.48.4` (VPS directo) | 🔴 Caído |

### 3.6 Reversión inmediata (política: cliente activo estable primero)

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

**Confirmación de recuperación:**
```bash
curl -k -m 5 -o /dev/null -w "%{http_code}\n" https://agc-conectachile.cl
# 200
```
✅ AGC restablecido en aprox. 15 minutos desde detección.

Las rich-rules de `cloudflare-ips` y el ipset se conservan (andamio para reintento futuro; no hacen daño en paralelo con el service blanket).

### 3.7 Investigación de causa raíz (post-reversión)

Se descartaron dos hipótesis iniciales tras cuestionamiento del operador:

- ❌ **"Faltó reiniciar Apache":** firewalld y Apache no se comunican; Apache siguió operando durante todo el candado (evidencia: vaulthost.cl siguió respondiendo perfecto).
- ❌ **"scm-pro.cl con SNI/certificado mal":** scm-pro.cl NO vive en este VPS — vive en el reseller SMART 10 (registro histórico del proyecto). No aparecía en `httpd.conf` por eso mismo. Estaba caído por la misma razón que los otros no-proxeados.

**Causa raíz confirmada (validada contra dashboard de Cloudflare del operador):**
> De los cuatro dominios que viven en este VPS (vaulthost, dagnstudio-vps, agc-conectachile, fluxyard-appsystem), **solo vaulthost.cl está bajo Cloudflare**. Los otros tres resuelven directo a `158.69.48.4`. El visitante que va a `agc-conectachile.cl` no pasa por Cloudflare — pega directo al VPS con su IP residencial normal. Esa IP no está en el ipset de Cloudflare → firewalld dropea → sitio caído. **No es un problema de Apache, ni de SNI, ni de certificados: es un problema de que el filtro estaba correctamente diseñado para "solo Cloudflare", pero solo un dominio está detrás de Cloudflare.**

Prueba en dashboard de Cloudflare de la cuenta: **"1 - 1 of 1 items"** — solo vaulthost.cl.

### 3.8 Descartes y correcciones honestas

Corrección explícita: **Cloudflare NO proxea MySQL (3306) ni el tráfico de AppSheet directo a la base de datos.** El plan Free solo cubre HTTP/HTTPS. La protección de la conexión AppSheet → MariaDB **es y seguirá siendo el ipset `appsheet-ips`** de SEC-02, independientemente de si `agc-conectachile.cl` entra a Cloudflare o no. Web y DB son caminos separados que hay que hardenear por separado.

### 3.9 Estado final Tarea 2 (revertido, con andamio conservado)

- [x] Sitio operativo (todos los dominios respondiendo)
- [x] Ipset `cloudflare-ips` con 15 entradas oficiales — **conservado**
- [x] Rich-rules 80/443 desde `cloudflare-ips` — **conservadas**
- [x] Services `http`/`https` en zona pública — **repuestos**
- [x] Bypass de origen — **temporalmente reabierto** (aceptable: exposición controlada mientras se planifica la migración de dominios)
- [x] Causa raíz identificada y validada contra el dashboard de Cloudflare
- [ ] Ticket cerrable solo cuando se completen los dominios activos en Cloudflare (dependencia externa, ver §5)

---

## 4. Hallazgos secundarios detectados durante la sesión (no resueltos)

Anotados para backlog, sin acción inmediata:

### 4.1 Puerto 3000/tcp abierto públicamente
```
LISTEN 0 511 *:3000 * users:(("node",pid=1450,fd=28))
```
Identificado como proceso Node de `acceso-camiones-api` en `/home/fluxyardappsystm/`, corriendo **como root** (no como el usuario `fluxyardappsystm`). Dos banderas:
1. Contradice el registro del proyecto que marca `fluxyardappsystm` como "congelada" — hay una app viva.
2. Corre como root, violando el principio de menor privilegio.

**Ticket futuro:** `Node-root — Investigar y re-scopear acceso-camiones-api`. No 3B.

### 4.2 Antipatrón de firewall vigente en SSH y paneles admin
`services: ... ssh` y `ports: 2083/tcp 2087/tcp 2096/tcp ...` en zona pública, junto con rich-rule de `10.8.0.0/24`. Mismo patrón resuelto para 3306 en SEC-02, pendiente para SSH y paneles → contenido de **SEC-12 y SEC-13**. Confirmado en vivo por el intento del operador de conectar a `158.69.48.4:2087` con VPN activa, que funcionó — porque no depende de la VPN todavía.

### 4.3 Certificado por IP interna
Al entrar a `https://10.8.0.1:2087` el navegador marca "No seguro" — es mismatch de nombre, no falla de cifrado. Aceptable operacionalmente; opcional agregar hostname interno + AutoSSL en el futuro.

---

## 5. Decisiones tomadas y no tomadas

| # | Decisión | Estado |
|---|---|---|
| D1 | WG-00: dar por válido el estado de WireGuard como pre-requisito de SEC-12/SEC-13 | ✅ Adoptada |
| D2 | Tarea 2: revertir ante caída de cliente activo (AGC) | ✅ Adoptada (política: cliente activo estable primero) |
| D3 | Conservar el andamio (ipset + rich-rules) para reintento futuro | ✅ Adoptada |
| D4 | No cerrar la Fase 3B hoy; el supuesto original ("todos los dominios detrás de Cloudflare") era falso | ✅ Adoptada |
| — | Reintentar candado 80/443 hoy sin resolver el mapa de dominios | ❌ Descartada |

---

## 6. Definition of Done — Sesión

- [x] WG-00 cerrado con evidencia bilateral (servidor + cliente)
- [x] Tarea 2 documentada como *"intentada + revertida + causa raíz identificada"* — no como fallo
- [x] Andamio técnico (ipset, rich-rules) conservado y documentado
- [x] Cliente activo (AGC) restablecido y verificado externamente (código HTTP 200)
- [x] Tres tickets nuevos abiertos para el backlog (ver §7 y documento de retrospectiva)

---

## 7. Próximos pasos

**Bloqueantes de Fase 3B (nuevos tickets):**
- `SEC-03B` — Migrar dominios activos al Cloudflare del operador (dagnstudio y AGC como prioritarios; requiere aviso previo a AGC como manda la regla del proyecto).
- `SEC-03C` — Decidir estado real de fluxyard-appsystem.cl (mantener / apagar / mover) antes de decidir si se protege.
- `Node-root` — Investigación del proceso Node de acceso-camiones-api corriendo como root en `fluxyardappsystm`.

**Independientes de Fase 3B (siguen ejecutables):**
- `SEC-13` — Restringir 2083/2087/2096 a VPN-only. Pre-validado por el éxito de WG-00. Bajo riesgo residual dado que la VPN funciona.
- `SEC-12` — SSH key-only. Requiere confirmar KVM de OVH como red de seguridad antes de tocar.

---

*Fin de la bitácora técnica — Sprint 2 · WG-00 + Tarea 2 · VaultHost.cl*
