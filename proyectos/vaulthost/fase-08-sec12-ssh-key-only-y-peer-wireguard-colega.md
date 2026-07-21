# Bitácora — SEC-12: SSH Key-Only Authentication + Peer WireGuard Colega AGC

> **Documento de bitácora de fase.** Registra lo ejecutado, los problemas, las decisiones y las lecciones de esta sesión.

| Campo | Valor |
|---|---|
| **Proyecto** | VaultHost VPS — hardening de infraestructura |
| **Ticket** | SEC-12: Deshabilitar SSH password authentication (forzar solo llave) |
| **Tipo de trabajo** | 🛠️ Modificador — cambios en SSH, firewall y WireGuard |
| **Responsable** | Oscar (owner, dev y administrador) |
| **Acompañamiento** | Claude (ingeniero de cabecera) |
| **Fecha** | 20-07-2026 |
| **Estado** | ✅ SEC-12 cerrado |
| **Dependencias previas** | SEC-02 cerrado (MariaDB 3306 restringido), WireGuard operativo (WG-00), VPN túnel funcional (10.8.0.1 ↔ 10.8.0.2) |

---

## Control de versiones del documento

| Versión | Fecha | Cambio | Autor |
|---|---|---|---|
| 1.0 | 20-07-2026 | Bitácora inicial de SEC-12 | Oscar + Claude |

---

## 1. Objetivo de la fase

Tres objetivos combinados en un solo ticket por cercanía técnica:

1. **Deshabilitar autenticación por contraseña en SSH** — forzar acceso exclusivamente por llave criptográfica (Ed25519).
2. **Restringir el puerto 22 (SSH) a VPN-only** — remover la apertura genérica que anulaba la rich-rule existente (mismo patrón hallado y corregido previamente en SEC-02 con el puerto 3306).
3. **Crear un peer WireGuard para el colega de AGC** — habilitar acceso VPN para Francis (colaborador técnico de AGC), requisito para que pueda acceder a cPanel cuando SEC-13 se ejecute.

---

## 2. Punto de partida

| Recurso | Estado |
|---|---|
| Acceso SSH actual | Password authentication (contraseña robusta +40 caracteres, pero compartida con WHM) |
| Puerto 22 en firewall | Abierto a todo internet vía `services: ssh` + rich-rule VPN-only (la rich-rule anulada por la apertura genérica) |
| Peers WireGuard | Solo 1: Oscar (10.8.0.2) |
| Usuario SSH | `almalinux` (usuario default de AlmaLinux/OVH, luego escalada con `sudo su`) |

### Hallazgo previo al ticket

Se detectó que la misma contraseña se usaba para SSH (`almalinux`) y WHM (`root`). Esto es un antipatrón de seguridad: un solo secreto comprometido abre dos puertas críticas. La migración a llave SSH elimina este riesgo para el acceso SSH, y se recomendó cambiar la contraseña de WHM a una distinta.

---

## 3. Pasos ejecutados

### Paso 1 — Decisión de nivel de seguridad para la llave

**Opciones evaluadas:**
- **Nivel 1:** llave Ed25519 de archivo + passphrase (simple, sin hardware adicional).
- **Nivel 2:** llave respaldada en YubiKey 4 vía PIV (más seguro pero requiere PuTTY-CAC o Pageant con PKCS#11).

**Decisión:** Nivel 1 (llave de archivo). La complejidad de configurar PIV con PuTTY en Windows no justifica el beneficio marginal en este momento. Se documentó como posible mejora futura (SEC-12b).

### Paso 2 — Generación de llave Ed25519 con PuTTYgen 🛠️

**En la máquina Windows de Oscar:**

1. Abrir PuTTYgen → seleccionar tipo **EdDSA** → curva **Ed25519**.
2. Click **Generate** → mover el mouse para entropía.
3. Configurar passphrase fuerte (almacenada separadamente en Bitwarden).
4. **Save private key** → archivo `.ppk` para PuTTY.
5. Copiar el texto del cuadro "Public key for pasting into OpenSSH authorized_keys file".

**Exportación adicional de la llave privada en formato OpenSSH:**
- PuTTYgen → Conversions → Export OpenSSH key → guardada como backup en Bitwarden.

**POR QUÉ el formato OpenSSH además del `.ppk`:** si algún día necesitas acceder desde Linux, macOS, o cualquier cliente que no sea PuTTY, el formato `.ppk` no sirve — OpenSSH es el formato universal. Tener ambos formatos respaldados cubre cualquier escenario de recuperación.

### Paso 3 — Instalar llave pública en el servidor 🛠️

```bash
# Como usuario almalinux en el VPS
mkdir -p /home/almalinux/.ssh
chmod 700 /home/almalinux/.ssh

# Pegar la llave pública copiada de PuTTYgen
echo "ssh-ed25519 AAAA... oscar-dagnstudio-vps-2026" >> /home/almalinux/.ssh/authorized_keys

chmod 600 /home/almalinux/.ssh/authorized_keys
chown -R almalinux:almalinux /home/almalinux/.ssh
```

**POR QUÉ los permisos importan:** OpenSSH rechaza silenciosamente las llaves si el directorio `.ssh` o el archivo `authorized_keys` tienen permisos demasiado abiertos (700 para el directorio, 600 para el archivo — ni más ni menos).

### Paso 4 — Configurar PuTTY + Pageant para usar la llave 🛠️

1. Abrir **Pageant** (agente de llaves de PuTTY).
2. **Add Key** → seleccionar el archivo `.ppk` generado → ingresar passphrase.
3. Pageant queda residente en la bandeja del sistema con la llave en memoria.
4. En PuTTY, configurar la sesión con:
   - Host: `10.8.0.1`, Port: `22`
   - Connection → Data → Auto-login username: `almalinux`
   - Connection → SSH → Auth → "Attempt authentication using Pageant": ✅ habilitado

### Paso 5 — Prueba de la llave ANTES de deshabilitar password ✅

**QUÉ:** abrir una nueva sesión PuTTY y verificar que entre por llave sin pedir password.

**Resultado:**
```
login as: almalinux
Authenticating with public key "oscar-dagnstudio-vps-2026" from agent
```

✅ Login exitoso por llave, Pageant proporcionó la llave automáticamente, sin prompt de password.

**POR QUÉ este orden importa:** si deshabilitáramos password antes de confirmar que la llave funciona, y la llave fallara por cualquier motivo (permiso, formato, path), quedaríamos bloqueados del servidor (salvo consola KVM de Nuthost).

### Paso 6 — Deshabilitar autenticación por password 🛠️

**Manteniendo la sesión existente abierta como red de seguridad:**

```bash
# Backup del archivo de configuración con fecha
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak-$(date +%Y%m%d)

# Aplicar el cambio
sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config

# Verificar visualmente
grep -i "^PasswordAuthentication" /etc/ssh/sshd_config
# Resultado: PasswordAuthentication no ✅

# Reiniciar el servicio
systemctl restart sshd
systemctl status sshd --no-pager
# Resultado: active (running) ✅
```

### Paso 7 — Validación post-cambio ✅

**Prueba 1 — Nueva sesión PuTTY vía VPN (debe funcionar):**
```
Authenticating with public key "oscar-dagnstudio-vps-2026" from agent
```
✅ Login exitoso por llave, después del restart de sshd.

**Prueba 2 — Prueba externa real desde Kali (fuera de la VPN, debe fallar):**
```bash
# Desde Kali (red externa, sin WireGuard)
ssh almalinux@158.69.48.4
# Resultado: timeout (conexión no llega — puerto 22 bloqueado por firewall a cualquier IP fuera de 10.8.0.0/24)
```
✅ Timeout confirmado — el puerto 22 no es alcanzable desde fuera de la VPN.

### Paso 8 — Restringir puerto 22 en el firewall 🛠️

**Hallazgo (mismo patrón que SEC-02 y luego SEC-13):** el puerto 22 tenía una rich-rule VPN-only, pero también estaba abierto genéricamente vía `services: ssh` en la zona `public`, anulando la restricción.

```bash
# Remover el servicio SSH de la apertura genérica
firewall-cmd --zone=public --remove-service=ssh --permanent
firewall-cmd --reload
```

La rich-rule VPN-only ya existente (`rule family="ipv4" source address="10.8.0.0/24" port port="22" protocol="tcp" accept`) quedó como el único camino de acceso al puerto 22.

### Paso 9 — Crear peer WireGuard para el colega AGC 🛠️

**POR QUÉ en este ticket:** SEC-13 (restringir cPanel/WHM/Webmail a VPN-only) requiere que el colega tenga acceso VPN antes de cerrar los puertos. Crear el peer ahora permite validar su conectividad antes de ejecutar SEC-13.

**Generación de llaves para el nuevo peer:**
```bash
wg genkey | tee /root/colega-privatekey | wg pubkey > /root/colega-publickey
cat /root/colega-publickey
```

**Registro del peer en caliente (respetando SaveConfig = true):**
```bash
# Agregar el peer en la interfaz VIVA (no editando el archivo)
wg set wg0 peer <llave-pública-del-colega> allowed-ips 10.8.0.3/32

# Guardar el estado actual al archivo de configuración
wg-quick save wg0

# Verificar
wg show
```

**POR QUÉ `wg set` + `wg-quick save` y no editar el archivo:**
El `wg0.conf` tiene `SaveConfig = true`. Esto significa que cuando se baja la interfaz (`wg-quick down`), WireGuard sobrescribe el archivo con el estado que tenía en memoria — pisando cualquier edición manual que hayas hecho. El flujo correcto es: modificar la interfaz en caliente con `wg set`, y luego persistir con `wg-quick save`.

**Preparación del archivo .conf para el colega:**
```ini
[Interface]
PrivateKey = <llave privada generada para el colega>
Address = 10.8.0.3/32
DNS = 1.1.1.1
MTU = 1280

[Peer]
PublicKey = 9snolisG9DxAA4hcdyqBPgEECBuSKBexkCuhpGjO3VY=
AllowedIPs = 10.8.0.0/24
Endpoint = 158.69.48.4:51820
PersistentKeepalive = 25
```

**Nota sobre MTU = 1280:** este valor se agregó posteriormente (durante SEC-13) al descubrir problemas de fragmentación de paquetes. Para la bitácora de SEC-12, el archivo original se envió sin esta línea — fue corregido en la sesión de SEC-13.

**Entrega al colega:** archivo `.conf` enviado para importar en la app WireGuard for Windows.

### Paso 10 — Respaldo de llaves en Bitwarden 🛠️

Se crearon las siguientes entradas en Bitwarden Desktop (carpeta "Soluciones Informáticas"):

| Entrada en Bitwarden | Contenido |
|---|---|
| **WireGuard — Servidor VPS (llave privada + pública)** | Llaves del servidor WireGuard |
| **WireGuard — Mi PC (llave privada + pública)** | Llaves del túnel personal de Oscar |
| **VaultHost VPS - llave privada Ed25519 (backup)** | Llave privada SSH en formato OpenSSH |
| **VaultHost VPS - Key passphrase Ed25519 (backup)** | Passphrase de la llave SSH |
| **VaultHost VPS - Instrucciones Recovery Open SSH** | Procedimiento paso a paso para restaurar acceso SSH desde cero |
| **Plantilla Reconstrucción WireGuard** | Template del `.conf` con placeholders para reconstruir el túnel |
| **Instrucciones de Reconstrucción WireGuard en Nuevo Equipo** | Guía para Windows y Linux |

---

## 4. Limpieza post-sesión

```bash
# Eliminar llaves temporales del colega del servidor
rm /root/colega-privatekey /root/colega-publickey
```

La llave privada del colega no debe quedar en el servidor — solo el colega la necesita.

---

## 5. Problemas encontrados y resolución

### P1 — Misma contraseña para SSH y WHM

**Síntoma:** la contraseña del usuario `almalinux` (SSH) era la misma que la de `root` en WHM.

**Causa:** no se había separado las credenciales de los dos servicios.

**Solución:** la migración a llave SSH elimina el uso de password para SSH enteramente. Se recomendó cambiar la contraseña de WHM a una distinta para cerrar el vector restante.

### P2 — Puerto 22 con apertura genérica anulando rich-rule (tercer hallazgo del mismo patrón)

**Síntoma:** `services: ssh` en la zona `public` de firewalld permitía acceso desde cualquier IP, a pesar de existir una rich-rule que lo restringía a `10.8.0.0/24`.

**Causa:** la misma que en SEC-02 (3306) y luego en SEC-13 (2083/2087/2096) — las aperturas genéricas de puerto/servicio anulan las rich-rules restrictivas.

**Solución:** `firewall-cmd --remove-service=ssh --permanent && firewall-cmd --reload`.

**Lección reforzada (tercera vez):** este es un patrón recurrente en firewalld — siempre verificar que no exista una apertura genérica coexistiendo con una rich-rule restrictiva.

---

## 6. Decisiones tomadas

| # | Decisión | Por qué |
|---|---|---|
| **D1** | Llave Ed25519 de archivo (no YubiKey PIV) | Menor complejidad, sin riesgo de bloqueo propio durante hardening activo. YubiKey queda como mejora futura (SEC-12b). |
| **D2** | Respaldo dual de la llave: formato `.ppk` (PuTTY) + OpenSSH (universal) | Cubre recuperación desde cualquier sistema operativo/cliente SSH. |
| **D3** | Peer del colega creado como parte de SEC-12 | Prerequisito técnico para SEC-13 (cPanel VPN-only). Agruparlo aquí evita un ticket adicional y mantiene la trazabilidad. |
| **D4** | `wg set` + `wg-quick save` (no editar archivo manual) | Única forma correcta de agregar peers con `SaveConfig = true` activo. |

---

## 7. Mapa de acceso post SEC-12

| Servicio | Método de autenticación | Restricción de red |
|---|---|---|
| SSH (22) | Solo llave Ed25519 (password deshabilitado) | Solo VPN (`10.8.0.0/24`) |
| WHM (2087) | Password (aún activo) | Solo VPN (rich-rule, pero blanket aún presente — corregido en SEC-13) |
| cPanel (2083) | Password | Público (cerrado en SEC-13) |

---

## 8. Lecciones aprendidas

1. **Nunca deshabilitar password authentication sin confirmar primero que la llave funciona.** El orden correcto es: instalar llave → probar llave → deshabilitar password → probar de nuevo. Mantener una sesión abierta como red de seguridad durante todo el proceso.

2. **El backup de configuración con fecha (`sshd_config.bak-20260720`) permite rollback preciso** sin adivinar cuándo se hizo el cambio.

3. **Los permisos de `.ssh/` y `authorized_keys` son críticos y silenciosos.** OpenSSH rechaza llaves sin mensaje de error visible si los permisos son incorrectos (`.ssh` debe ser 700, `authorized_keys` debe ser 600).

4. **Pageant simplifica el flujo diario:** carga la llave una vez al iniciar Windows, todas las sesiones PuTTY se autentican automáticamente sin repetir la passphrase.

5. **`SaveConfig = true` en WireGuard cambia completamente el flujo de administración.** Editar el archivo a mano mientras la interfaz está activa es una receta para perder cambios. Siempre usar `wg set` en caliente + `wg-quick save`.

6. **La prueba externa real (desde fuera de la VPN) es la validación definitiva**, no los comandos read-only del servidor. Se usó Kali en red externa para confirmar que SSH no era alcanzable desde internet.

---

## 9. English practice — glosario de la fase

| English | Español | Uso en esta fase |
|---|---|---|
| Key-based authentication | Autenticación por llave | Método que reemplazó el password para SSH |
| Passphrase | Frase de contraseña | Cifrado que protege la llave privada en disco |
| Key agent (Pageant) | Agente de llaves | Software que mantiene la llave descifrada en memoria |
| Authorized keys | Llaves autorizadas | Archivo del servidor con las llaves públicas aceptadas |
| Password authentication | Autenticación por contraseña | Método deshabilitado en esta fase |
| Split tunnel | Túnel dividido | Config de WireGuard donde solo cierto tráfico pasa por la VPN |
| Peer | Par/compañero | Cada extremo de un túnel WireGuard |
| SaveConfig | Guardar configuración | Directiva de WireGuard que auto-guarda el estado al bajar la interfaz |

> *Practice:* "Always confirm that key-based authentication works before disabling password authentication — keep an active session open as a safety net during the transition."
> *Práctica:* "Siempre confirmar que la autenticación por llave funciona antes de deshabilitar la autenticación por contraseña — mantener una sesión activa abierta como red de seguridad durante la transición."

---

## 10. Definition of Done — SEC-12

- [x] Llave Ed25519 generada con PuTTYgen
- [x] Llave pública instalada en `/home/almalinux/.ssh/authorized_keys`
- [x] Login por llave validado vía PuTTY + Pageant (sin prompt de password)
- [x] `PasswordAuthentication no` aplicado en `sshd_config`
- [x] `sshd` reiniciado y funcionando correctamente
- [x] Puerto 22 removido de apertura genérica (`services: ssh` eliminado)
- [x] Rich-rule VPN-only (`10.8.0.0/24`) como único acceso a SSH
- [x] Prueba externa real desde Kali: timeout confirmado (SSH no alcanzable desde internet)
- [x] Prueba interna real desde PuTTY vía VPN: login por llave confirmado post-restart
- [x] Peer WireGuard del colega AGC creado (10.8.0.3/32)
- [x] Respaldo completo en Bitwarden (llave privada OpenSSH + `.ppk` + passphrase + instrucciones de recovery)
- [x] Llaves temporales del colega eliminadas del servidor

---

## 11. Próximo paso (ejecutado el mismo día)

**SEC-13:** Restringir puertos 2083/2087/2096 a VPN-only — pendiente hasta que el colega confirme handshake exitoso. La confirmación se obtuvo el mismo día y SEC-13 se ejecutó inmediatamente después (ver bitácora separada de SEC-13).

---

*Fin de la bitácora — SEC-12 · VaultHost.cl*
