# Bitácora — Sprint 2 / Fase 3B (parte 1): Verificación WireGuard + Intento de candado Cloudflare

> **Documento de bitácora de fase — capa de ejecución.**
> Registra lo que se ejecutó, en qué orden, con qué output y con qué resultado.
> Su documento hermano `fase-05-reflexion-fase-3b-parte-1.md` cubre el análisis, los aprendizajes y la evolución del diseño.

| Campo | Valor |
|---|---|
| **Proyecto** | VaultHost VPS — hardening de infraestructura |
| **Sprint / Fase** | Sprint 2 — Fase 3B, sesión 1 |
| **Tickets abordados** | `WG-00` (verificación WireGuard) · `Tarea 2` (candar 80/443 a IPs Cloudflare) |
| **Tipo de trabajo** | 🔍 read-only (WG-00) + 🛠️ modificador con reversión (Tarea 2) |
| **Responsable** | Oscar |
| **Acompañamiento** | Claude (ingeniero de cabecera) |
| **Fecha** | 09-07-2026 |
| **Estado** | ✅ WG-00 cerrado · ⚠️ Tarea 2 revertida, causa raíz identificada, tickets nuevos abiertos |

---

## 1. Objetivo original de la sesión

Cerrar la Fase 3B completa: candar el origen del VPS a IPs de Cloudflare, restringir paneles admin a VPN, y endurecer SSH. Verificando primero que el túnel WireGuard (prerequisito de todo lo demás) esté operativo.

**Resultado real:** se cerró WG-00 con éxito. La Tarea 2 (candar 80/443) reveló un supuesto oculto en el diseño que forzó reversión y replanificación. SEC-12 y SEC-13 quedan para próxima sesión.

---

## 2. Punto de partida

| Recurso | Estado al inicio |
|---|---|
| WireGuard servidor | Existencia asumida por presencia de la subred `10.8.0.0/24` en rich-rules; sin confirmación explícita |
| WireGuard Windows 11 | Existencia recordada por Oscar, sin registro en bitácoras previas |
| Firewall 80/443 | Abierto a todo internet vía servicios `http`/`https` de firewalld |
| Cloudflare | Proxeando `vaulthost.cl` (confirmado por whatsmydns.net previamente) |
| Puerto 3000 | Abierto públicamente (arrastrado desde troubleshooting del Ticket 3, sin cerrar) |

---

## 3. WG-00 — Verificación WireGuard (🔍 read-only)

### 3.1 Lado servidor

**Ritual de apertura:**
```bash
whoami; hostname; uptime
```
Sesión abierta como `almalinux`, escalada a root con `sudo su`.

**Comando 1 — Estado de la interfaz WireGuard:**
```bash
sudo wg show
```
```
interface: wg0
  public key: 9snolisG9DxAA4hcdyqBPgEECBuSKBexkCuhpGjO3VY=
  private key: (hidden)
  listening port: 51820

peer: SueSMiclgrn9LxZOaPnM/TTfb3PYl0gNWfuU05o5Wis=
  endpoint: 190.21.129.60:61764
  allowed ips: 10.8.0.2/32
```

**Comando 2 — IP de la interfaz interna:**
```bash
ip a show wg0
```
```
3: wg0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 ...
    inet 10.8.0.1/24 scope global wg0
```

**Comando 3 — Archivos y servicio:**
```bash
sudo ls -la /etc/wireguard/
systemctl status wg-quick@wg0
```
- `server_private.key`, `server_public.key`, `wg0.conf` — permisos `600` (`rw-------`)
- Servicio `wg-quick@wg0.service`: `Loaded: enabled`, `Active: active (exited)` desde 2026-07-09 10:27:07 UTC
- Nota en log: `RTNETLINK answers: Network is unreachable` en un proceso hijo durante el arranque. Interfaz igualmente `UP,LOWER_UP` según `ip a`.

**Comando 4 — Reglas firewall confirmadas:**
```bash
sudo firewall-cmd --list-all
```
Rich-rules ya existentes:
```
rule family="ipv4" source address="10.8.0.0/24" port port="3306" protocol="tcp" accept
rule family="ipv4" source address="10.8.0.0/24" port port="22"   protocol="tcp" accept
rule family="ipv4" source ipset="appsheet-ips"  port port="3306" protocol="tcp" accept
rule family="ipv4" source address="10.8.0.0/24" port port="2087" protocol="tcp" accept
```

### 3.2 Lado cliente Windows 11

Ventana de WireGuard verificada:

| Campo | Valor |
|---|---|
| Túnel | "VPS" |
| Estado (pre-activación) | Inactivo |
| Direcciones (peer local) | `10.8.0.2/24` |
| IPs permitidas (AllowedIPs) | **`10.8.0.1/32`** |
| Endpoint | `158.69.48.4:51820` |
| Persistent keepalive | 25 |

### 3.3 Activación y prueba de handshake

Túnel activado desde la app de Windows. Verificación post-activación:

```
peer: SueSMiclgrn9LxZOaPnM/TTfb3PYl0gNWfuU05o5Wis=
  endpoint: 190.21.142.89:64905
  allowed ips: 10.8.0.2/32
  latest handshake: 1 minute, 50 seconds ago
  transfer: 308 B received, 92 B sent
```

Windows reportó en paralelo:
- Estado: **Activo**
- Último saludo: hace 25 segundos
- Transferir: 184 B recibidos, 552 B enviados

Prueba de acceso a WHM: `https://10.8.0.1:2087` respondió correctamente desde el navegador con la VPN activa (advertencia esperada de certificado por hostname/IP mismatch, no de cifrado).

**Definition of Done WG-00 — cumplida ✅**
- [x] WireGuard servidor instalado, activo, con servicio `enabled`
- [x] Cliente Windows configurado con `AllowedIPs = 10.8.0.1/32` (split-tunnel estricto)
- [x] Handshake criptográfico confirmado en ambos extremos
- [x] Contadores de transferencia consistentes con tráfico real
- [x] Acceso a WHM por IP interna del túnel (`10.8.0.1:2087`) confirmado

---

## 4. Puerto 3000 — investigación (🔍 read-only, sin acción)

**Motivación:** el puerto aparecía en la zona pública de firewalld sin restricción de origen, presuntamente arrastrado desde el Ticket 3.

**Comando:**
```bash
sudo ss -tlnp | grep ':3000'
```
```
LISTEN 0 511 *:3000 *:* users:(("node",pid=1450,fd=28))
```

**Identificación del proceso:**
```bash
ps -p 1450 -o pid,ppid,user,lstart,cmd
ls -la /proc/1450/cwd
```
```
CMD:  /usr/bin/node --require .../fluxyardappsystm/acceso-camiones-api/...
USER: root
cwd:  /home/fluxyardappsystm/acceso-camiones-api
```

**Hallazgo:** proceso Node vivo, corriendo como `root`, propiedad de la cuenta cPanel `fluxyardappsystm` (marcada como "congelada" en el registro del proyecto). Contradice el estado documentado y viola principio de menor privilegio. **Decisión de la sesión: no tocar hoy, abrir ticket dedicado.**

---

## 5. Tarea 2 — Candar 80/443 a IPs de Cloudflare (🛠️)

### 5.1 Fase constructiva

**Creación del ipset:**
```bash
firewall-cmd --permanent --new-ipset=cloudflare-ips --type=hash:net
for ip in 103.21.244.0/22 103.22.200.0/22 103.31.4.0/22 \
          104.16.0.0/13 104.24.0.0/14 108.162.192.0/18 \
          131.0.72.0/22 141.101.64.0/18 162.158.0.0/15 \
          172.64.0.0/13 173.245.48.0/20 188.114.96.0/20 \
          190.93.240.0/20 197.234.240.0/22 198.41.128.0/17; do
    firewall-cmd --permanent --ipset=cloudflare-ips --add-entry=$ip
done
firewall-cmd --reload
```
Fuente autoritativa: `cloudflare.com/ips` (IPv4, 15 rangos vigentes al 09-07-2026).

**Rich-rules agregadas:**
```bash
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source ipset="cloudflare-ips" port port="80"  protocol="tcp" accept'
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source ipset="cloudflare-ips" port port="443" protocol="tcp" accept'
firewall-cmd --reload
```

**Checkpoint 1 (verificación en paralelo):** las 15 entradas confirmadas en el ipset; sitio `https://vaulthost.cl` respondiendo normalmente. Rich-rules operativas junto al servicio `https` blanket todavía presente. ✅

### 5.2 Fase destructiva (candar)

**Remoción del acceso blanket:**
```bash
firewall-cmd --permanent --remove-service=http
firewall-cmd --permanent --remove-service=https
firewall-cmd --reload
```

### 5.3 Checkpoint 2 — validación del candado

**Perspectiva externa desde Kali (VirtualBox NAT):**
```bash
curl -k -m 5 https://158.69.48.4
# curl: (28) Connection timed out after 5002 milliseconds  ← bypass cerrado ✅

curl -k -m 5 https://vaulthost.cl
# HTTP 200 con HTML completo ← camino vía Cloudflare intacto ✅
```

**Observación crítica en el mismo checkpoint:**
- `dagnstudio-vps.cl` reportado como caído por Oscar al abrirlo en navegador durante la validación.

### 5.4 Investigación en vivo — mapa DNS real

Batch `dig` desde Kali (con corrección de shell — `→` unicode reemplazado por `->`):

```bash
for d in vaulthost.cl scm-pro.cl agc-conectachile.cl dagnstudio-vps.cl fluxyard-appsystem.cl; do
  echo -n "$d -> HTTP "
  curl -k -m 5 -o /dev/null -w "%{http_code} (resuelve: %{remote_ip})\n" https://$d
done
```

Post-reversión (ver §5.5), resultado con firewall abierto:

| Dominio | HTTP | remote_ip | Interpretación |
|---|---|---|---|
| vaulthost.cl | 200 | `104.21.7.38` | Proxeado por Cloudflare |
| scm-pro.cl | 200 | `104.21.36.34` | Proxeado por Cloudflare, **pero apuntando a origen distinto (SMART 10 reseller)** |
| agc-conectachile.cl | 200 | `158.69.48.4` | Directo al VPS, sin Cloudflare |
| dagnstudio-vps.cl | 200 | `158.69.48.4` | Directo al VPS, sin Cloudflare |
| fluxyard-appsystem.cl | 200 | `158.69.48.4` | Directo al VPS, sin Cloudflare |

### 5.5 Reversión

**Contexto:** `agc-conectachile.cl` es cliente activo con regla explícita del proyecto de "aviso previo antes de cualquier acción disruptiva". Con AGC caído, la reversión se ejecuta inmediatamente.

```bash
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https
firewall-cmd --reload
```

**Validación post-reversión:**
```bash
curl -k -m 5 -o /dev/null -w "%{http_code}\n" https://agc-conectachile.cl
# 200 ← AGC recuperado ✅
```

### 5.6 Confirmación por dashboard de Cloudflare

Captura del dashboard `dash.cloudflare.com/domains`: filtro "Domains", 1 - 1 of 1 items → **solo `vaulthost.cl` está registrado en la cuenta Cloudflare de VaultHost**. Los otros 3 dominios que viven en el VPS (`dagnstudio-vps.cl`, `agc-conectachile.cl`, `fluxyard-appsystem.cl`) no están ahí, por eso el candado por `cloudflare-ips` los deja fuera.

---

## 6. Estado del andamio dejado en el firewall

**Se conservan (no se revierten):**
- ipset `cloudflare-ips` con 15 rangos IPv4 oficiales.
- Rich-rules `accept` para 80 y 443 desde `cloudflare-ips`.

**Motivo:** son andamio ya construido y validado para la próxima ronda. Cuando los dominios pendientes estén migrados a Cloudflare, cerrar el candado será una sola operación de dos comandos.

**Se revirtió:**
- Servicios `http` y `https` re-agregados a la zona pública (estado original).

---

## 7. Definition of Done de la sesión

- [x] WG-00 cerrado con evidencia completa (handshake + acceso a WHM por VPN)
- [x] Puerto 3000 identificado y anotado como ticket futuro
- [x] Tarea 2 ejecutada, causa raíz de la caída de dominios identificada, reversión limpia
- [x] Estado del firewall al cierre: idéntico al de apertura salvo por el andamio de Cloudflare
- [x] Los 5 vhosts respondiendo (validado con curl externo)
- [ ] ⚠️ Fase 3B NO cerrada — bloqueada por SEC-03B (migración de dominios a Cloudflare)

---

## 8. Herramientas y componentes utilizados en la sesión

| Herramienta | Uso en esta sesión |
|---|---|
| PuTTY | Sesión SSH al VPS como `almalinux` |
| WireGuard (Windows 11) | Cliente peer con split-tunnel `AllowedIPs=10.8.0.1/32` |
| WireGuard (AlmaLinux) | Servidor peer `wg0`, escuchando en UDP/51820 |
| WHM (por VPN) | Verificación de acceso vía `https://10.8.0.1:2087` |
| firewalld + nftables | Ipsets y rich-rules; `firewall-cmd` para operar |
| Kali Linux en VirtualBox (modo NAT) | Perspectiva externa autoritativa |
| curl + dig | Mediciones read-only end-to-end |
| Cloudflare (dashboard) | Fuente de verdad del inventario de dominios proxeados |
| whatsmydns.net (previamente) | Confirmación de propagación DNS de Cloudflare |
| Claude Desktop | Guía técnica, verificación cruzada, generación de artefactos |
| Mermaid.js | Diagramas de arquitectura (antes/después) y secuencia |
| DiagramGPT (Eraser.io) | Diagrama de arquitectura ANTES (paga por créditos, luego migrado a Mermaid) |
| Trello Kanban | Gestión de tickets del sprint |

---

## 9. Próximos pasos técnicos

**Nuevos tickets abiertos a raíz de esta sesión:**
- `SEC-03B` — Migrar `dagnstudio-vps.cl` y `agc-conectachile.cl` a Cloudflare (bloqueante de cerrar Tarea 2).
- `SEC-03C` — Decidir destino de `fluxyard-appsystem.cl` (dominio + cuenta congelada).
- `Node-root` — Investigar y re-scopear proceso `pid=1450` (`acceso-camiones-api`) corriendo como root desde cuenta "congelada".

**Tickets ya en backlog que siguen pendientes:**
- `SEC-12` — SSH key-only (deshabilitar `PasswordAuthentication`).
- `SEC-13` — Restringir 2083/2087/2096 a VPN-only.
- `SEC-09` — fail2ban.
- `SEC-05/07` — HSTS/CSP + ModSecurity + rate-limit webhook.

---

*Fin de la bitácora de ejecución — Fase 3B parte 1 · VaultHost.cl*
