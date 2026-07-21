# VaultHost VPS — Guía de Configuración, Respaldo y Recuperación: WireGuard + SSH/PuTTY

> **Documento operacional confidencial.** Contiene referencias a llaves criptográficas y procedimientos de recuperación de acceso al VPS.
> Almacenar en repositorio **privado** de GitHub. Las llaves reales viven en Bitwarden, no en este documento.

| Campo | Valor |
|---|---|
| **Proyecto** | VaultHost VPS — acceso seguro |
| **Responsable** | Oscar (owner, administrador) |
| **Fecha de creación** | 20-07-2026 |
| **Última actualización** | 20-07-2026 |
| **Servidor** | VPS OVH — `vps-7a76ae17` — IP pública `158.69.48.4` |
| **SO** | AlmaLinux 9.8 — cPanel/WHM |

---

## Control de versiones

| Versión | Fecha | Cambio | Autor |
|---|---|---|---|
| 1.0 | 20-07-2026 | Versión inicial con estado completo post SEC-12/SEC-13 | Oscar + Claude |

---

## 1. Arquitectura de acceso — visión general

```
┌──────────────────────────────────────────────────────────────┐
│                        INTERNET                               │
│                                                               │
│   Oscar (Mi PC)              Colega AGC (Francis)             │
│   ┌──────────────┐           ┌──────────────┐                │
│   │ WireGuard    │           │ WireGuard    │                │
│   │ 10.8.0.2/32  │           │ 10.8.0.3/32  │                │
│   │ MTU: 1280    │           │ MTU: 1280    │                │
│   └──────┬───────┘           └──────┬───────┘                │
│          │ UDP:51820                 │ UDP:51820               │
│          │ (cifrado WireGuard)       │ (cifrado WireGuard)    │
│          └───────────┬───────────────┘                        │
│                      │                                        │
│                      ▼                                        │
│            ┌─────────────────────┐                            │
│            │ VPS — 158.69.48.4   │                            │
│            │ wg0: 10.8.0.1/24    │                            │
│            │ Puerto: 51820/udp   │                            │
│            ├─────────────────────┤                            │
│            │ Servicios internos: │                            │
│            │  SSH   → :22        │ ← solo 10.8.0.0/24        │
│            │  WHM   → :2087      │ ← solo 10.8.0.0/24        │
│            │  cPanel→ :2083      │ ← solo 10.8.0.0/24        │
│            │  Webmail→:2096      │ ← solo 10.8.0.0/24        │
│            └─────────────────────┘                            │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. WireGuard — Configuración del servidor (VPS)

### Archivo de configuración: `/etc/wireguard/wg0.conf`

```ini
[Interface]
Address = 10.8.0.1/24
SaveConfig = true
ListenPort = 51820
PrivateKey = <ver Bitwarden: "WireGuard — Servidor VPS (llave privada + pública)">

[Peer]
# Oscar — Mi PC
PublicKey = <ver Bitwarden: "WireGuard — Mi PC (llave privada + pública)" → Llave Pública>
AllowedIPs = 10.8.0.2/32

[Peer]
# Colega AGC — Francis
PublicKey = <ver Bitwarden: llave pública del colega>
AllowedIPs = 10.8.0.3/32
```

### Datos clave del servidor

| Parámetro | Valor |
|---|---|
| IP interna WireGuard | `10.8.0.1/24` |
| Puerto de escucha | `51820/udp` |
| MTU de la interfaz wg0 | `1420` (default, no modificado) |
| Servicio systemd | `wg-quick@wg0.service` (enabled, arranca automáticamente en cada boot) |
| Archivo de config | `/etc/wireguard/wg0.conf` |
| Regla SaveConfig | `true` — significa que los cambios en caliente se guardan al archivo cuando se baja la interfaz |

### Llaves del servidor — ubicación en Bitwarden

| Dato | Entrada en Bitwarden |
|---|---|
| Llave privada del servidor | **"WireGuard — Servidor VPS (llave privada + pública)"** → campo "Llave Privada" |
| Llave pública del servidor | **"WireGuard — Servidor VPS (llave privada + pública)"** → campo "Llave Pública" |

> **Llave pública del servidor** (esta sí es segura de documentar aquí, es información pública por diseño):
> `9snolisG9DxAA4hcdyqBPgEECBuSKBexkCuhpGjO3VY=`

### Regla crítica sobre SaveConfig = true

> **Nunca editar `/etc/wireguard/wg0.conf` a mano mientras `wg0` está activo.**
> Cuando `SaveConfig = true`, al hacer `wg-quick down wg0`, WireGuard **sobrescribe** el archivo con el estado en memoria (que no incluye tus ediciones manuales). El flujo correcto es:
>
> **Para agregar peers:** usar `wg set wg0 peer <pubkey> allowed-ips <ip>` en caliente, luego `wg-quick save wg0`.
>
> **Para editar configuración:** hacer `wg-quick down wg0` primero (¡nunca desde SSH que depende del túnel!), editar el archivo, luego `wg-quick up wg0`.

---

## 3. WireGuard — Configuración de clientes

### 3.1 Cliente: Oscar (Mi PC) — IP `10.8.0.2`

**Archivo de configuración en la app WireGuard Windows:**

```ini
[Interface]
PrivateKey = <ver Bitwarden: "WireGuard — Mi PC (llave privada + pública)" → Llave Interfaz>
Address = 10.8.0.2/24
MTU = 1280

[Peer]
PublicKey = 9snolisG9DxAA4hcdyqBPgEECBuSKBexkCuhpGjO3VY=
AllowedIPs = 10.8.0.0/24
Endpoint = 158.69.48.4:51820
PersistentKeepalive = 25
```

**Llaves en Bitwarden:**

| Dato | Entrada en Bitwarden |
|---|---|
| Llave privada (interfaz) | **"WireGuard — Mi PC (llave privada + pública)"** → campo "Llave Interfaz" |
| Llave pública | **"WireGuard — Mi PC (llave privada + pública)"** → campo "Llave Pública" |

**Nombre del túnel en la app:** `VPS`

### 3.2 Cliente: Colega AGC — Francis — IP `10.8.0.3`

**Archivo de configuración en la app WireGuard Windows:**

```ini
[Interface]
PrivateKey = <proporcionada al colega al momento de creación del peer>
Address = 10.8.0.3/32
DNS = 1.1.1.1
MTU = 1280

[Peer]
PublicKey = 9snolisG9DxAA4hcdyqBPgEECBuSKBexkCuhpGjO3VY=
AllowedIPs = 10.8.0.0/24
Endpoint = 158.69.48.4:51820
PersistentKeepalive = 25
```

**Nombre del túnel en la app:** `AGC-VPN`

**Nota:** la llave privada del colega fue generada en el servidor y entregada al momento de la creación del peer. Si se pierde, se debe generar un nuevo par de llaves y reconfigurar el peer (ver §7).

---

## 4. Mapa de IPs asignadas dentro del túnel

| IP WireGuard | Asignado a | Rol | Estado |
|---|---|---|---|
| `10.8.0.1` | Servidor VPS | Servidor WireGuard | ✅ Activo |
| `10.8.0.2` | Oscar (Mi PC) | Admin principal | ✅ Activo |
| `10.8.0.3` | Colega AGC (Francis) | Colaborador cPanel AGC | ✅ Activo |
| `10.8.0.4` — `10.8.0.254` | Sin asignar | Disponibles para futuros peers | — |

---

## 5. SSH / PuTTY — Configuración y llaves

### Datos de conexión SSH

| Parámetro | Valor |
|---|---|
| Host | `10.8.0.1` (requiere VPN WireGuard activa) |
| Puerto | `22` |
| Usuario | `almalinux` |
| Autenticación | Solo llave SSH (password deshabilitado — SEC-12) |
| Tipo de llave | Ed25519 |

### Llaves SSH — ubicación en Bitwarden

| Dato | Entrada en Bitwarden |
|---|---|
| Llave privada (formato OpenSSH) | **"VaultHost VPS - llave privada Ed25519 (backup)"** |
| Passphrase de la llave | **"VaultHost VPS - Key passphrase Ed25519 (backup)"** |
| Instrucciones de recuperación | **"VaultHost VPS - Instrucciones Recovery Open SSH"** |
| Fingerprint de verificación | `SHA256:O64NrLJ2a0ne3Z/hrp0w4Duz3x/kX6+uKnmvWePUjPI` |

### Configuración PuTTY

| Parámetro | Valor |
|---|---|
| Session → Host Name | `10.8.0.1` |
| Session → Port | `22` |
| Connection → Data → Auto-login username | `almalinux` |
| Connection → SSH → Auth → Private key file | Ruta al archivo `.ppk` generado desde la llave OpenSSH |
| Connection → SSH → Auth → Passphrase | Se ingresa al conectar (o vía Pageant pre-cargado) |

### Formato de la llave para PuTTY

PuTTY usa formato `.ppk`, no el formato OpenSSH directo. Para convertir:

1. Abrir **PuTTYgen**.
2. **Conversions** → **Import key** → seleccionar el archivo de llave privada OpenSSH.
3. Ingresar la passphrase cuando la pida.
4. **Save private key** → genera el archivo `.ppk`.

Alternativamente, cargar la llave en **Pageant** (agente de llaves de PuTTY) para no tener que ingresar la passphrase en cada conexión.

---

## 6. Procedimientos de recuperación

### 6.1 Escenario: Pérdida de acceso SSH + VPN funcional

**Síntoma:** PuTTY no conecta, pero la VPN WireGuard está activa y funcional.

**Causa probable:** llave SSH corrupta, passphrase olvidada, o archivo `.ppk` perdido.

**Procedimiento:**
1. Abrir Bitwarden Desktop.
2. Buscar **"VaultHost VPS - llave privada Ed25519 (backup)"** — copiar el contenido completo.
3. Pegar en un archivo de texto plano → Guardar como `recovery.key` (Tipo: Todos los archivos).
4. Abrir **PuTTYgen** → **Conversions** → **Import key** → seleccionar `recovery.key`.
5. Ingresar la passphrase desde Bitwarden: **"VaultHost VPS - Key passphrase Ed25519 (backup)"**.
6. **Save private key** → genera nuevo `.ppk`.
7. Cargar en Pageant o configurar directamente en PuTTY → Connection → SSH → Auth.
8. Verificar que el fingerprint coincide: `SHA256:O64NrLJ2a0ne3Z/hrp0w4Duz3x/kX6+uKnmvWePUjPI`.
9. Conectar a `10.8.0.1:22` como usuario `almalinux`.

### 6.2 Escenario: Pérdida de acceso VPN (túnel no conecta)

**Síntoma:** WireGuard muestra "Activo" pero no hay handshake, o no se puede activar el túnel.

**Procedimiento — reconstruir el túnel en la misma máquina:**
1. Abrir WireGuard como Administrador.
2. Eliminar el túnel existente (botón `X` o "Eliminar túnel").
3. Clic en **"Agregar túnel"** → **"Add empty tunnel..."**
4. Pegar el contenido de la plantilla (ver Bitwarden: **"Plantilla Reconstrucción WireGuard"**), reemplazando los placeholders con las llaves reales desde Bitwarden.
5. **Guardar** → **Activar**.

**Plantilla de referencia (con placeholders):**

```ini
[Interface]
PrivateKey = <Llave Interfaz — de tu nota "WireGuard — Mi PC">
Address = 10.8.0.2/24
MTU = 1280

[Peer]
PublicKey = 9snolisG9DxAA4hcdyqBPgEECBuSKBexkCuhpGjO3VY=
Endpoint = 158.69.48.4:51820
AllowedIPs = 10.8.0.0/24
PersistentKeepalive = 25
```

> **Nota sobre MTU = 1280:** esta línea es obligatoria. Sin ella, las páginas HTTPS pesadas (cPanel, WHM) no cargarán en la mayoría de conexiones de ISPs chilenos por problemas de Path MTU Discovery.

### 6.3 Escenario: Configurar WireGuard en un equipo nuevo

**Procedimiento para Windows:**
1. Descargar cliente oficial: `https://www.wireguard.com/install/`
2. Instalar (next-next-finish).
3. Abrir la app → **"Add Tunnel"** → **"Add empty tunnel..."**
4. Pegar el contenido de la plantilla (§6.2), reemplazando la `PrivateKey` con la llave correcta de Bitwarden.
5. **Guardar** → **Activar**.

**Procedimiento para Linux:**
```bash
sudo apt install wireguard   # o dnf/yum según la distro
sudo nano /etc/wireguard/wg0.conf   # pegar la plantilla
sudo wg-quick up wg0
```

**Si es un equipo completamente nuevo (nuevo peer, no migración):** se necesita generar un nuevo par de llaves y registrar el peer en el servidor (ver §7).

### 6.4 Escenario: VPN y SSH perdidos — no hay acceso al servidor

**Síntoma:** no puedes conectar ni por VPN ni por SSH. El servidor sigue corriendo pero no tienes forma de entrar.

**Procedimiento:**
1. Abrir navegador → ir al panel de Nuthost (portal de cliente donde gestionas/pagas el VPS).
2. Buscar tu servicio VPS.
3. **Opción A (preferida):** usar el botón **"Console"** o **"KVM/VNC"** — da acceso directo como si estuvieras frente al servidor.
4. **Opción B (si no hay consola):** usar el botón **"Reboot"** — reinicia el VPS. El servicio `wg-quick@wg0.service` está habilitado y levanta el túnel automáticamente al arrancar.
5. Después del reboot, esperar ~60 segundos y probar la conexión WireGuard + PuTTY normalmente.

> **IMPORTANTE:** nunca usar "Power Off" salvo emergencia extrema — un apagado no levanta los servicios automáticamente como un reboot.

### 6.5 Escenario: Pérdida total de Bitwarden (todas las llaves perdidas)

**Esto es el peor caso — nivel desastre.** Si se pierden todas las llaves de Bitwarden:

1. Acceder al servidor vía **consola KVM de Nuthost** (no requiere llaves — es acceso directo al terminal).
2. Login como `root` (si se tiene la contraseña de root del VPS que otorgó Nuthost al momento de la contratación).
3. Generar nuevas llaves WireGuard:
   ```bash
   wg genkey | tee /root/nueva-privatekey | wg pubkey > /root/nueva-publickey
   ```
4. Editar `/etc/wireguard/wg0.conf` con la nueva llave privada del servidor.
5. Generar nuevo par de llaves SSH:
   ```bash
   ssh-keygen -t ed25519 -C "recovery" -f /home/almalinux/.ssh/id_ed25519_new
   cat /home/almalinux/.ssh/id_ed25519_new.pub >> /home/almalinux/.ssh/authorized_keys
   ```
6. Copiar las nuevas llaves privadas a una nueva bóveda de Bitwarden.
7. Reconstruir los clientes WireGuard en cada máquina con las nuevas llaves.

---

## 7. Procedimiento: Agregar un nuevo peer WireGuard

Para dar acceso VPN a un nuevo colaborador o equipo:

### Paso 1 — Generar llaves para el nuevo peer (en el servidor)

```bash
wg genkey | tee /root/nuevo-peer-privatekey | wg pubkey > /root/nuevo-peer-publickey
cat /root/nuevo-peer-publickey
# → copiar esta llave pública
```

### Paso 2 — Registrar el peer en el servidor (en caliente)

```bash
# Elegir la siguiente IP disponible del mapa (§4)
wg set wg0 peer <llave-pública-del-nuevo-peer> allowed-ips 10.8.0.X/32

# Guardar el estado al archivo (por SaveConfig = true)
wg-quick save wg0

# Verificar
wg show
```

### Paso 3 — Preparar la configuración del cliente

Crear un archivo `.conf` para el nuevo peer:

```ini
[Interface]
PrivateKey = <llave privada generada en Paso 1>
Address = 10.8.0.X/32
MTU = 1280

[Peer]
PublicKey = 9snolisG9DxAA4hcdyqBPgEECBuSKBexkCuhpGjO3VY=
Endpoint = 158.69.48.4:51820
AllowedIPs = 10.8.0.0/24
PersistentKeepalive = 25
```

### Paso 4 — Entregar al colaborador

Enviar la configuración de forma segura (nunca por email sin cifrar). Opciones:
- Dictarla por llamada.
- Bitwarden Send (temporal, con expiración).
- Presencialmente (el más seguro).

### Paso 5 — Limpiar llaves del servidor

```bash
rm /root/nuevo-peer-privatekey /root/nuevo-peer-publickey
```

La llave privada del peer **no debe quedar en el servidor** — solo el peer la necesita.

### Paso 6 — Actualizar el mapa de IPs (§4 de este documento)

Agregar el nuevo peer al mapa con su IP, nombre y rol asignado.

---

## 8. Procedimiento: Revocar acceso a un peer

Si un colaborador deja el proyecto o su equipo es comprometido:

```bash
# Remover el peer (usar la llave pública del peer a revocar)
wg set wg0 peer <llave-pública-del-peer> remove

# Guardar
wg-quick save wg0

# Verificar
wg show
```

El cambio es inmediato — el peer pierde acceso en el acto.

---

## 9. URLs de acceso — referencia rápida (bookmarks)

| Servicio | URL | Requiere |
|---|---|---|
| WHM (administración) | `https://10.8.0.1:2087` | VPN activa |
| cPanel (cuenta agcapp) | `https://10.8.0.1:2083` | VPN activa |
| cPanel (cuenta vaulthostweb) | `https://10.8.0.1:2083` | VPN activa |
| Webmail | `https://10.8.0.1:2096` | VPN activa |
| SSH (PuTTY) | `10.8.0.1:22` | VPN activa + llave SSH |
| Panel Nuthost (emergencia) | URL del portal de cliente Nuthost | Solo cuenta Nuthost |

> **Advertencia de certificado:** al acceder por IP `10.8.0.1`, el navegador mostrará `ERR_CERT_COMMON_NAME_INVALID`. Esto es esperado y seguro — la conexión es HTTPS cifrada dentro de un túnel WireGuard ya cifrado. Clic en "Avanzado" → "Continuar".

---

## 10. Checklist de verificación post-recuperación

Después de cualquier procedimiento de recuperación (§6), verificar:

- [ ] VPN WireGuard activa — `wg show` muestra handshake reciente
- [ ] SSH funcional — PuTTY conecta a `10.8.0.1:22` con llave
- [ ] WHM accesible — `https://10.8.0.1:2087` carga el login
- [ ] cPanel accesible — `https://10.8.0.1:2083` carga el login
- [ ] Webmail accesible — `https://10.8.0.1:2096` carga el login
- [ ] Sitios web públicos operativos — `https://vaulthost.cl` y `https://agc-conectachile.cl` cargan normalmente
- [ ] Correo funcional — enviar/recibir un correo de prueba

---

## 11. English practice — glosario

| English | Español | Uso en este documento |
|---|---|---|
| Peer | Par | Cada extremo de un túnel WireGuard |
| Private key | Llave privada | Secreto que identifica un peer (nunca compartir) |
| Public key | Llave pública | Derivada de la privada, segura de compartir |
| Allowed IPs | IPs permitidas | Qué tráfico puede pasar por el túnel para este peer |
| Endpoint | Punto de conexión | IP:puerto del servidor WireGuard |
| Keepalive | Mantener vivo | Paquete periódico para mantener el túnel activo detrás de NAT |
| Key rotation | Rotación de llaves | Cambiar periódicamente las llaves por seguridad |
| Revoke | Revocar | Eliminar acceso de un peer |
| KVM console | Consola KVM | Acceso directo al servidor sin depender de la red |
| Passphrase | Frase de contraseña | Contraseña que protege una llave privada |
| Fingerprint | Huella digital | Identificador único de una llave para verificación |

> *Practice:* "When onboarding a new peer, generate the key pair on the server, register the public key with wg set, and deliver the private key securely — then delete it from the server."
> *Práctica:* "Al incorporar un nuevo peer, generar el par de llaves en el servidor, registrar la llave pública con wg set, y entregar la llave privada de forma segura — luego eliminarla del servidor."

---

*Fin del documento — WireGuard + SSH/PuTTY Configuración, Respaldo y Recuperación · VaultHost.cl*
