# Bitácora — SEC-13: Restringir puertos WHM/cPanel/Webmail (2083/2087/2096) a VPN-only

> **Documento de bitácora de fase.** Registra lo ejecutado, los problemas, las decisiones y las lecciones de esta sesión.

| Campo | Valor |
|---|---|
| **Proyecto** | VaultHost VPS — hardening de infraestructura |
| **Ticket** | SEC-13: Restringir puertos WHM/cPanel/Webmail a VPN-only |
| **Tipo de trabajo** | 🛠️ Modificador — cambios en firewall + diagnóstico de conectividad VPN |
| **Responsable** | Oscar (owner, dev y administrador) |
| **Acompañamiento** | Claude (ingeniero de cabecera) |
| **Fecha** | 20-07-2026 |
| **Estado** | ✅ SEC-13 cerrado |
| **Dependencias previas** | SEC-03 cerrado (Cloudflare + firewall candado), SEC-12 cerrado (SSH key-only), peer WireGuard del colega AGC creado y validado |

---

## Control de versiones del documento

| Versión | Fecha | Cambio | Autor |
|---|---|---|---|
| 1.0 | 20-07-2026 | Bitácora inicial de SEC-13 | Oscar + Claude |

---

## 1. Objetivo de la fase

Restringir el acceso a los puertos de administración de cPanel (2083), WHM (2087) y Webmail (2096) para que solo sean alcanzables desde la red VPN WireGuard (`10.8.0.0/24`), eliminando la exposición pública de estas interfaces de gestión.

**Prerrequisito validado antes de ejecutar:** el colega de AGC (peer `10.8.0.3`) confirmó handshake exitoso y conectividad funcional vía WireGuard — condición que estaba pendiente desde la creación del peer en SEC-12.

---

## 2. Punto de partida

### Estado del firewall antes de SEC-13

```
public (active)
  interfaces: eth0
  ports: 2083/tcp 2096/tcp 25/tcp 587/tcp 465/tcp 993/tcp 995/tcp 51820/udp 2087/tcp
  rich rules:
    rule family="ipv4" source address="10.8.0.0/24" port port="22" protocol="tcp" accept
    rule family="ipv4" source ipset="appsheet-ips" port port="3306" protocol="tcp" accept
    rule family="ipv4" source ipset="cloudflare-ips" port port="80" protocol="tcp" accept
    rule family="ipv4" source address="10.8.0.0/24" port port="2087" protocol="tcp" accept
    rule family="ipv4" source ipset="cloudflare-ips" port port="443" protocol="tcp" accept
    rule family="ipv4" source address="10.8.0.0/24" port port="3306" protocol="tcp" accept
```

**Observación crítica del punto de partida:**

- **2083 (cPanel) y 2096 (Webmail):** abiertos a todo internet vía `ports:` genérico, sin ninguna restricción.
- **2087 (WHM):** tenía una rich-rule VPN-only **pero también seguía en `ports:` genérico** — la rich-rule estaba siendo anulada por la apertura pública (hallazgo de la sesión, ver §5).

### Peers WireGuard activos

| Peer | IP interna | Rol |
|---|---|---|
| Oscar (Mi PC) | `10.8.0.2/32` | Admin principal |
| Colega AGC (Francis) | `10.8.0.3/32` | Colaborador — acceso a cPanel de `agcapp` |

### Confirmación de handshake del colega (prerrequisito)

```bash
wg show
# peer: F+TpZq894+WQrrP9ZeVum+fvijt7iPLo/QldE2SdiR4=
#   endpoint: 200.119.231.240:64050
#   allowed ips: 10.8.0.3/32
#   latest handshake: 39 minutes, 56 seconds ago
#   transfer: 68.59 KiB received, 251.40 KiB sent
```

✅ Handshake reciente y tráfico real bidireccional — condición de ejecución cumplida.

---

## 3. Pasos ejecutados

### Paso 1 — Restringir 2083 (cPanel) y 2096 (Webmail) 🛠️

```bash
# Quitar los puertos de la apertura genérica pública
firewall-cmd --zone=public --remove-port=2083/tcp --permanent
firewall-cmd --zone=public --remove-port=2096/tcp --permanent

# Agregar rich-rules VPN-only
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="10.8.0.0/24" port port="2083" protocol="tcp" accept' --permanent
firewall-cmd --zone=public --add-rich-rule='rule family="ipv4" source address="10.8.0.0/24" port port="2096" protocol="tcp" accept' --permanent

# Aplicar
firewall-cmd --reload
```

✅ Resultado: `success` en los 5 comandos.

### Paso 2 — Hallazgo y corrección del 2087 (WHM) 🛠️

**Hallazgo:** al revisar el output de `firewall-cmd --list-all` post-Paso 1, se detectó que el 2087 seguía en la lista `ports:` genérica. Esto significaba que la rich-rule VPN-only que ya existía para 2087 **nunca estuvo realmente restringiendo nada** — la apertura genérica la anulaba silenciosamente.

**Lección documentada previamente que se aplicó aquí:**
> *"Rich-rule allowlists are nullified by blanket port openings in the public zone — must remove blanket rules after implementing ipset-based allowlists."*

```bash
# Quitar el 2087 de la apertura genérica pública
firewall-cmd --zone=public --remove-port=2087/tcp --permanent
firewall-cmd --reload
```

✅ Resultado: `success`.

### Paso 3 — Validación del firewall post-cambios ✅

```bash
firewall-cmd --list-all
```

**Estado final del firewall:**

```
public (active)
  interfaces: eth0
  ports: 25/tcp 587/tcp 465/tcp 993/tcp 995/tcp 51820/udp
  rich rules:
    rule family="ipv4" source ipset="cloudflare-ips" port port="80" protocol="tcp" accept
    rule family="ipv4" source ipset="cloudflare-ips" port port="443" protocol="tcp" accept
    rule family="ipv4" source address="10.8.0.0/24" port port="22" protocol="tcp" accept
    rule family="ipv4" source address="10.8.0.0/24" port port="2096" protocol="tcp" accept
    rule family="ipv4" source address="10.8.0.0/24" port port="2083" protocol="tcp" accept
    rule family="ipv4" source address="10.8.0.0/24" port port="2087" protocol="tcp" accept
    rule family="ipv4" source ipset="appsheet-ips" port port="3306" protocol="tcp" accept
    rule family="ipv4" source address="10.8.0.0/24" port port="3306" protocol="tcp" accept
```

✅ Los puertos 2083, 2087 y 2096 ya NO están en `ports:` y solo existen como rich-rules con source `10.8.0.0/24`.

---

## 4. Diagnóstico y resolución de problemas de conectividad

Después de aplicar los cambios, ambos usuarios (Oscar y el colega) experimentaron problemas para acceder a cPanel/WHM. El diagnóstico fue extenso y se documenta en detalle.

### Problema 1 — Acceso por hostname público en vez de IP interna

**Síntoma:** Oscar intentó acceder a WHM via `vps-7a76ae17.vps.ovh.ca:2087` y obtuvo timeout.

**Causa:** ese hostname resuelve a `158.69.48.4` (IP pública del VPS), que está fuera del rango `10.8.0.0/24`. El tráfico salía por internet normal (no por el túnel WireGuard) y el firewall lo descartaba correctamente.

**Solución:** usar siempre `https://10.8.0.1:2087` (IP interna del servidor dentro del túnel WireGuard).

### Problema 2 — Colega accedía por dominio sin puerto explícito

**Síntoma:** colega intentaba acceder a `cpanel.agc-conectachile.cl` (sin `:2083`). Timeout.

**Causa:** sin puerto explícito, el navegador intenta puerto 443. El puerto 443 está restringido a Cloudflare-only (SEC-03). El dominio resuelve a la IP pública (`158.69.48.4`) vía DNS, no a `10.8.0.1`.

**Solución:** acceder siempre con puerto explícito y por la IP interna: `https://10.8.0.1:2083`.

### Problema 3 — MTU: páginas que no cargan a pesar de conectividad TCP OK

**Síntoma:** `Test-NetConnection -Port 2087` reportaba `TcpTestSucceeded: True`, pero la página de WHM no cargaba (timeout visual en el navegador).

**Diagnóstico paso a paso:**

1. Se verificó que `wg0` estaba en zona "no zone" de firewalld (no asignada a `public`), pero se descartó como causa porque el tráfico fluía correctamente por el túnel.

2. `curl -vk https://10.8.0.1:2087` **desde el propio servidor** respondió `200 OK` con la página completa (39547 bytes) — descartando problema del servicio.

3. **Prueba de MTU desde Windows (PowerShell):**
   ```powershell
   ping 10.8.0.1 -l 1400 -f
   # "Es necesario fragmentar el paquete pero se especificó DF."
   # 100% pérdida
   ```
   Esto confirmó un "agujero negro de MTU" (Path MTU Discovery roto): los paquetes grandes no caben en algún tramo de la ruta y el ICMP "fragmentación necesaria" de vuelta está bloqueado por algún firewall intermedio.

4. **Lógica del problema:** la interfaz `wg0` del servidor tenía `mtu 1420`. El handshake TLS de cPanel/WHM genera paquetes grandes (cadena de certificados Let's Encrypt). Los paquetes chicos (TCP SYN/ACK, keepalive WireGuard) pasaban sin problema; los grandes se perdían silenciosamente.

**Solución aplicada — lado cliente:** agregar `MTU = 1280` en la sección `[Interface]` del archivo de configuración WireGuard de cada cliente Windows:

```ini
[Interface]
PrivateKey = <sin cambios>
Address = 10.8.0.x/24
MTU = 1280

[Peer]
...
```

**Valor 1280:** mínimo garantizado por el estándar IPv6 — todo camino de internet moderno lo soporta sin fragmentar. Seguro y universal.

**Verificación post-fix:**
```powershell
ping 10.8.0.1 -l 1252 -f
# Respuesta desde 10.8.0.1: bytes=1252 tiempo=157ms TTL=64
# 0% perdidos ✅
```

### Problema 4 — `wg-quick down wg0` cortó la sesión SSH

**Síntoma:** al intentar persistir el cambio de MTU del servidor, se ejecutó `wg-quick down wg0` desde una sesión SSH que dependía del propio túnel WireGuard. La sesión se cortó inmediatamente.

**Recuperación:** acceso vía panel de Nuthost (proveedor del VPS) → botón "Reboot" → el VPS reinició y `wg-quick@wg0.service` levantó el túnel automáticamente.

**Lección crítica documentada:**
> **Nunca ejecutar `wg-quick down wg0` desde una sesión SSH que pasa por el propio túnel.** Es como serruchar la rama en la que estás sentado. Para operaciones que requieran bajar el túnel, usar la consola KVM/VNC del proveedor (Nuthost), o encadenar `down` y `up` en un solo comando para minimizar la ventana de desconexión.

### Problema 5 — Confusión de consolas: Linux vs Windows

**Síntoma:** se corrieron comandos de diagnóstico de Windows (como `tracert`, `ping -f`) en la sesión SSH de Linux, produciendo resultados incorrectos o errores.

**Detalle:** en Linux, `ping -f` significa **flood ping** (no "Don't Fragment" como en Windows). Se dispararon 2.3 millones de paquetes en modo flood contra el loopback, generando un falso 43% de "pérdida" que era solo ruido. `tracert` no existe en Linux (se llama `traceroute`).

**Lección:** siempre tener claridad sobre en qué máquina se está ejecutando cada comando diagnóstico. Los comandos de red tienen sintaxis y semántica distinta entre Windows y Linux.

---

## 5. Hallazgos importantes

### Hallazgo 1 — El 2087 (WHM) nunca estuvo realmente restringido

La rich-rule VPN-only para 2087 existía desde antes de esta sesión, pero coexistía con una apertura genérica en `ports:` que la anulaba. **WHM estuvo accesible públicamente todo el tiempo que se creyó protegido.** Esto se corrigió en el Paso 2.

### Hallazgo 2 — MTU como requisito operacional para clientes WireGuard

El MTU default de WireGuard (1420) causa problemas de carga de páginas HTTPS pesadas en rutas con Path MTU Discovery roto (muy común en ISPs chilenos). Todo cliente WireGuard que se conecte al VPS debe configurar `MTU = 1280` en su `[Interface]`.

### Hallazgo 3 — El MTU solo necesita ajustarse del lado cliente

Después del reboot del servidor (que revirtió el MTU temporal de `wg0` a 1420), ambos clientes (Oscar y colega) accedieron sin problemas usando `MTU = 1280` solo en sus clientes Windows. El TCP negocia un MSS (Maximum Segment Size) más chico basado en el MTU del cliente, y el servidor respeta ese MSS en sus respuestas. No es necesario modificar el MTU del servidor.

### Hallazgo 4 — Ping servidor→colega falla (ICMP bloqueado por Windows)

```bash
ping -c 5 10.8.0.3
# 5 packets transmitted, 0 received, 100% packet loss
```

Esto ocurre pese a tener handshake WireGuard activo y tráfico TCP/HTTP funcional. **Causa:** Windows Defender Firewall bloquea echo-request ICMP entrante por defecto. No es un problema real de conectividad — solo afecta la herramienta `ping` como diagnóstico, no el tráfico real de aplicaciones.

---

## 6. Decisiones tomadas

| # | Decisión | Por qué |
|---|---|---|
| **D1** | Acceso a cPanel/WHM/Webmail exclusivamente vía `10.8.0.1:puerto` | Los dominios resuelven a la IP pública, que ya no acepta estos puertos. La IP interna VPN es el único camino. |
| **D2** | `MTU = 1280` como requisito para todo cliente WireGuard | Resuelve el problema de PMTUD roto sin efectos secundarios perceptibles. |
| **D3** | No modificar el MTU del servidor (dejarlo en 1420) | El fix del lado cliente es suficiente; menos cambios en el servidor = menos riesgo. |
| **D4** | Usar consola KVM/Reboot de Nuthost para emergencias de túnel | Alternativa cuando la sesión SSH depende del propio túnel que se necesita modificar. |

---

## 7. Estado final del firewall — mapa completo de puertos

### Puertos abiertos al público (sin restricción de origen)

| Puerto | Servicio | Notas |
|---|---|---|
| 25/tcp | SMTP | Correo saliente |
| 587/tcp | SMTP submission | Correo con autenticación |
| 465/tcp | SMTPS | Correo cifrado |
| 993/tcp | IMAPS | Correo IMAP cifrado |
| 995/tcp | POP3S | Correo POP3 cifrado |
| 51820/udp | WireGuard | Endpoint del túnel VPN |

### Puertos restringidos por origen (rich-rules)

| Puerto | Servicio | Origen permitido | Tipo de restricción |
|---|---|---|---|
| 80/tcp | HTTP | ipset `cloudflare-ips` | Solo Cloudflare (SEC-03) |
| 443/tcp | HTTPS | ipset `cloudflare-ips` | Solo Cloudflare (SEC-03) |
| 22/tcp | SSH | `10.8.0.0/24` | Solo VPN (SEC-12) |
| 2083/tcp | cPanel | `10.8.0.0/24` | Solo VPN (SEC-13) |
| 2087/tcp | WHM | `10.8.0.0/24` | Solo VPN (SEC-13) |
| 2096/tcp | Webmail | `10.8.0.0/24` | Solo VPN (SEC-13) |
| 3306/tcp | MariaDB | ipset `appsheet-ips` + `10.8.0.0/24` | AppSheet + VPN |

### Puertos completamente cerrados (no accesibles desde ningún origen externo)

Todo lo que no está listado arriba — incluyendo 80/443 directo (solo vía Cloudflare), 3000 (eliminado en SEC-02), etc.

---

## 8. Reglas operativas nuevas (post SEC-13)

### Para acceder a los paneles de administración

| Servicio | URL correcta | Requiere |
|---|---|---|
| WHM | `https://10.8.0.1:2087` | VPN WireGuard activa |
| cPanel | `https://10.8.0.1:2083` | VPN WireGuard activa |
| Webmail | `https://10.8.0.1:2096` | VPN WireGuard activa |
| SSH | `ssh almalinux@10.8.0.1` | VPN WireGuard activa + llave SSH |

### Advertencia de certificado esperada

Al acceder por IP `10.8.0.1`, el navegador mostrará `ERR_CERT_COMMON_NAME_INVALID` (el certificado SSL está emitido para el dominio, no para la IP). Esto es cosmético — la conexión sigue siendo HTTPS cifrada dentro de un túnel WireGuard ya cifrado (doble capa). Hacer clic en "Avanzado" → "Continuar" es seguro en este contexto.

### Para onboarding de nuevos peers WireGuard

Todo nuevo peer debe incluir `MTU = 1280` en su `[Interface]` del archivo de configuración del cliente. Esto es requisito operacional, no opcional.

---

## 9. Modelo de aprendizaje

### Analogía del edificio

**Antes (pre SEC-13):** el edificio (VPS) tenía la puerta del lobby (cPanel login) directamente en la calle. Cualquier persona del planeta podía caminar hasta ahí y probar contraseñas. El guardia (cPHulk) paraba a los que fallaban muchas veces, pero la puerta seguía visible y accesible.

**Después (post SEC-13):** la puerta del lobby ahora está dentro de un pasillo privado (túnel WireGuard). Para siquiera ver la puerta, necesitas la llave del pasillo (llave privada WireGuard correcta configurada en tu máquina). Sin esa llave, el pasillo no existe — ni siquiera sabes que hay una puerta ahí.

### Diagrama de flujo del acceso post SEC-13

```
ACCESO POR IP PÚBLICA (158.69.48.4:2083):
  Usuario → Internet → llega a IP pública → firewall: ¿viene de 10.8.0.0/24?
                                                          → NO → DROP (timeout)
                                                          → (nunca será SÍ por esta ruta)

ACCESO POR DOMINIO (cpanel.agc-conectachile.cl:2083):
  Usuario → DNS resuelve a 158.69.48.4 → misma ruta que arriba → DROP

ACCESO CORRECTO (10.8.0.1:2083 con VPN activa):
  Usuario → WireGuard → túnel cifrado → llega como 10.8.0.x → firewall: ¿viene de 10.8.0.0/24?
                                                                           → SÍ → ACCEPT → cPanel login ✅
```

---

## 10. Lecciones aprendidas

1. **Las rich-rules son anuladas por aperturas genéricas de `ports:`.** No basta con agregar una rich-rule restrictiva — hay que remover también la apertura genérica del puerto. Esto se confirmó empíricamente con el hallazgo del 2087.

2. **Nunca ejecutar `wg-quick down` desde una sesión que depende del túnel.** Usar consola KVM/VNC del proveedor o encadenar `down && up` para minimizar desconexión.

3. **El MTU de WireGuard (1420 default) causa problemas de carga de páginas HTTPS en rutas con PMTUD roto.** `MTU = 1280` en el cliente es la solución universal y segura.

4. **Distinguir claramente en qué máquina se ejecuta cada comando.** `ping -f` en Linux es flood, en Windows es Don't Fragment — semántica completamente opuesta.

5. **El `ping` no es indicador confiable de conectividad hacia máquinas Windows.** Windows Defender bloquea ICMP entrante por defecto. Usar `Test-NetConnection` (PowerShell) o verificar `wg show` (handshake + transfer) como fuentes de verdad.

6. **El panel del proveedor (Nuthost) es la salida de emergencia.** Siempre tener acceso al panel del proveedor para poder reiniciar o acceder por consola KVM cuando se pierde conectividad SSH/VPN.

---

## 11. English practice — glosario de la fase

| English | Español | Uso en esta fase |
|---|---|---|
| Blanket rule | Regla genérica/amplia | La apertura de puerto en `ports:` que anulaba la rich-rule |
| Rich rule | Regla enriquecida | Regla de firewalld con condiciones (source, port, protocol) |
| Path MTU Discovery (PMTUD) | Descubrimiento de MTU de ruta | Mecanismo que negocia el tamaño máximo de paquete entre dos hosts |
| MTU black hole | Agujero negro de MTU | Cuando PMTUD falla y los paquetes grandes se pierden silenciosamente |
| Maximum Segment Size (MSS) | Tamaño máximo de segmento | El TCP negocia esto basado en el MTU; determina el tamaño real de los datos |
| KVM / VNC console | Consola KVM/VNC | Acceso directo al servidor como si tuvieras monitor y teclado físico |
| Flood ping | Ping de inundación | `ping -f` en Linux — dispara paquetes a máxima velocidad |
| Don't Fragment (DF) | No fragmentar | `ping -f` en Windows — prohíbe fragmentación para diagnosticar MTU |
| Peer | Par/compañero | Cada extremo de un túnel WireGuard |
| Handshake | Apretón de manos | Negociación inicial de un túnel WireGuard o conexión TLS |

> *Practice:* "Rich rules are nullified by blanket port openings — always remove the generic port entry after adding a source-restricted rich rule, or the firewall allows traffic from any source."
> *Práctica:* "Las rich rules son anuladas por aperturas genéricas de puerto — siempre hay que remover la entrada genérica después de agregar una rich rule restringida por origen, o el firewall permitirá tráfico de cualquier fuente."

---

## 12. Definition of Done — SEC-13

- [x] Puerto 2083 (cPanel) restringido a VPN-only (`10.8.0.0/24`)
- [x] Puerto 2096 (Webmail) restringido a VPN-only (`10.8.0.0/24`)
- [x] Puerto 2087 (WHM) — blanket rule removida, ahora exclusivamente VPN-only
- [x] Validación: Oscar accede a WHM por `10.8.0.1:2087` con VPN activa ✅
- [x] Validación: colega AGC accede a cPanel por `10.8.0.1:2083` con VPN activa ✅
- [x] MTU = 1280 configurado y validado en ambos clientes Windows
- [x] Regla operativa documentada: todo nuevo peer WireGuard debe incluir `MTU = 1280`
- [x] Procedimiento de recuperación de emergencia validado (reboot vía panel Nuthost)

---

## 13. Tickets del checklist de hardening — estado actualizado

| Ticket | Estado | Notas |
|---|---|---|
| SEC-03 | ✅ Cerrado (19-07-2026) | Cloudflare proxy + firewall candado (80/443 Cloudflare-only) |
| SEC-04 | ⏳ Backlog | TLS/SSL check (testssl.sh) |
| SEC-05 | ⏳ Backlog | Headers HSTS + CSP |
| SEC-06 | ⏳ Backlog | OWASP ZAP passive scan |
| SEC-07 | ⏳ Backlog | ModSecurity (WAF) + rate-limit webhook |
| SEC-08 | ⏳ Backlog | Nikto liviano + banners |
| SEC-09 | ⏳ Backlog | fail2ban (SSH + login cPanel) |
| SEC-10 | ⚠️ Pendiente quick-win | Verificar shell de `vaulthostweb` restringida (deuda desde Ticket 3 del Sprint 1) |
| SEC-11 | ⏳ Backlog | Consolidación: bitácora fase-04-hardening |
| SEC-12 | ✅ Cerrado (20-07-2026) | SSH password authentication deshabilitado + key-only |
| SEC-13 | ✅ Cerrado (20-07-2026) | **Esta bitácora** — puertos 2083/2087/2096 VPN-only |

---

*Fin de la bitácora — SEC-13 · VaultHost.cl*
