# Bitácora — Sprint 1 / Ticket 3: Application Manager + Deploy en Producción

> **Documento de bitácora de fase.** Registra lo ejecutado, los problemas, las decisiones y las lecciones de esta fase del proyecto VaultHost.cl.
> Complementa al runbook oficial (*VaultHost — Arquitectura y Runbook de Despliegue*).

| Campo | Valor |
|---|---|
| **Proyecto** | VaultHost VPS — sitio comercial vaulthost.cl |
| **Sprint / Fase** | Sprint 1 (Despliegue) — **Ticket 3**: Application Manager + deploy |
| **Tipo de trabajo** | 🛠️ Modificador — primer deploy real de la app Next.js |
| **Responsable** | Oscar (owner, dev y administrador) |
| **Acompañamiento** | Claude (ingeniero de cabecera) |
| **Fecha** | 04-06-2026 |
| **Estado** | ✅ Ticket 3 cerrado · ⏭️ Prueba e2e MercadoPago pendiente (Ticket 4) |
| **Método de despliegue** | Opción A — cPanel Application Manager + Phusion Passenger |

---

## Control de versiones del documento

| Versión | Fecha | Cambio | Autor |
|---|---|---|---|
| 1.0 | 04-06-2026 | Bitácora inicial del Ticket 3 (deploy + troubleshooting) | Oscar + Claude |

---

## 1. Objetivo de la fase

Registrar la app Next.js en el Application Manager de cPanel, hacer el build de producción y dejar el sitio `vaulthost.cl` respondiendo correctamente con HTTPS, todas las rutas y el webhook de MercadoPago configurado.

---

## 2. Punto de partida

| Recurso | Estado |
|---|---|
| ea-nodejs22 + Passenger | ✅ Instalados (Ticket 2) |
| Repo en servidor | ❌ No clonado aún |
| Shell de `vaulthostweb` | ❌ Deshabilitado (noshell) |
| Application Manager | ✅ Feature habilitada |
| Sitio en producción | ❌ Pendiente |

---

## 3. Pasos ejecutados

### 3.1 — Habilitar shell temporalmente 🛠️

**QUÉ:** la cuenta `vaulthostweb` tenía shell deshabilitado (`noshell`), lo que impedía hacer `su - vaulthostweb` para clonar el repo y buildear como el usuario correcto.

**POR QUÉ importa el usuario:** los archivos deben pertenecer a `vaulthostweb`, no a `root`. Si se clona como root, los permisos quedan mal y Passenger no puede leer los archivos.

**COMANDO:**
```bash
whmapi1 modifyacct user=vaulthostweb shell=/bin/bash
```

> ⚠️ **Pendiente hardening:** restringir de vuelta a `/bin/false` o `/usr/local/cpanel/bin/noshell` al terminar el Sprint 1.

---

### 3.2 — Configurar deploy key SSH 🛠️

**QUÉ:** el repo en GitHub es privado. Git pidió autenticación al intentar clonar con HTTPS.

**POR QUÉ no funciona password:** GitHub eliminó la autenticación por contraseña en 2021. Se requiere token o SSH.

**Solución:** generar llave SSH en el servidor y registrarla como deploy key en GitHub (read-only).

```bash
ssh-keygen -t ed25519 -C "vaulthost-deploy" -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub  # → copiar a GitHub → repo → Settings → Deploy keys
```

**Verificación:**
```bash
ssh -T git@github.com
# Hi devopsterrible-creator/vaulthost-web! You've successfully authenticated...
```

---

### 3.3 — Clonar repo y preparar código 🛠️

```bash
su - vaulthostweb
cd /home/vaulthostweb
git clone git@github.com:devopsterrible-creator/vaulthost-web.git vaulthost-web
cd vaulthost-web
```

**Correcciones antes del build:**

```bash
# WhatsApp placeholder → número real (56971456955)
sed -i 's/wa\.me\/56900000000/wa.me\/56971456955/g' \
  app/checkout/failure/page.tsx \
  app/checkout/pending/page.tsx \
  app/contacto/page.tsx

# Agregar output standalone al next.config.ts
cat > next.config.ts << 'EOF'
import type { NextConfig } from "next";
const nextConfig: NextConfig = {
  output: 'standalone',
};
export default nextConfig;
EOF

# Crear .env.production (nunca en el repo — verificado en .gitignore)
cat > .env.production << 'EOF'
MP_ACCESS_TOKEN=...
NEXT_PUBLIC_MP_PUBLIC_KEY=...
BASE_URL=https://vaulthost.cl
NEXT_PUBLIC_BASE_URL=https://vaulthost.cl
EOF
```

---

### 3.4 — Build de producción 🛠️

```bash
npm ci && npm run build
```

**Output esperado (y obtenido):**
```
✓ Compiled successfully in 7.5s
✓ Finished TypeScript in 7.2s
✓ Generating static pages (15/15)
Route (app)
├ ○ /
├ ƒ /api/create-preference
├ ƒ /api/webhook
└ ○ /planes  ... (todas las rutas)
```

**Copiar estáticos** (el standalone no los mueve automáticamente):
```bash
cp -r public .next/standalone/public
cp -r .next/static .next/standalone/.next/static
```

**Verificar server.js:**
```bash
ls .next/standalone/server.js  # debe existir
```

---

### 3.5 — Registrar en Application Manager 🛠️

En cPanel de `vaulthostweb` → Software → Application Manager → Register Application:

| Campo | Valor |
|---|---|
| Application Name | `vaulthost-web` |
| Deployment Domain | `vaulthost.cl` |
| Base Application URL | `/` |
| Application Path | `vaulthost-web/.next/standalone` |
| Deployment Environment | `Production` |

Variables de entorno agregadas desde la interfaz (+ Add Variable):
- `BASE_URL`, `NEXT_PUBLIC_BASE_URL`, `MP_ACCESS_TOKEN`, `NEXT_PUBLIC_MP_PUBLIC_KEY`

→ **Deploy** → Status: **Enabled** ✅

---

### 3.6 — Troubleshooting: Index of / → 403 → solución final 🛠️

Esta fue la parte más larga del ticket. Se documenta con detalle porque es una lección técnica importante.

*(Ver sección §5 — Modelo de aprendizaje para la explicación completa con analogías.)*

**Secuencia de errores:**

| Error | Causa | Intento |
|---|---|---|
| `Index of /` | Apache sirve `public_html` estático, Passenger no activa | `.htaccess` con directivas Passenger |
| `500 Internal Server Error` | `PassengerEnabled` no puede ir en `.htaccess` | Mover config al `.conf` del virtualhost |
| `403 Forbidden` | `DocumentRoot` apunta a `public_html`, no al `standalone` | Symlink `public_html` → `standalone` |
| `Index of /` (con archivos correctos) | Apache sirve el directorio, no ejecuta `server.js` | Agregar `PassengerAppType node` + `PassengerStartupFile` |
| ✅ **Sitio funcionando** | Config completa con `app.js` wrapper | Solución final (ver abajo) |

**Solución final — el problema raíz:**

El `server.js` de Next.js standalone llama a `startServer()` con `port: currentPort`. Passenger en modo Apache no le pasa un `PORT` — usa un **socket Unix interno**. Sin `PassengerAppType node` explícito, Passenger no sabía que debía interceptar la app Node.

**Fix:**

1. Crear symlink `public_html` → `standalone`:
```bash
mv /home/vaulthostweb/public_html /home/vaulthostweb/public_html_bak
ln -s /home/vaulthostweb/vaulthost-web/.next/standalone /home/vaulthostweb/public_html
```

2. Crear `app.js` wrapper en el standalone:
```bash
cat > /home/vaulthostweb/public_html/app.js << 'EOF'
'use strict'
const path = require('path')
const dir = path.join(__dirname)
process.env.NODE_ENV = 'production'
process.chdir(__dirname)
process.env.PORT = process.env.PORT || 3000
require('./server.js')
EOF
```

3. Actualizar `.conf` del virtualhost con las directivas correctas:
```apache
<IfModule mod_passenger.c>
    PassengerEnabled on
    PassengerAppRoot "/home/vaulthostweb/public_html"
    PassengerStartupFile app.js
    PassengerAppType node
    PassengerNodejs /opt/cpanel/ea-nodejs22/bin/node
    PassengerAppEnv production
</IfModule>
<IfModule mod_env.c>
    SetEnv PORT 3000
    SetEnv BASE_URL "https://vaulthost.cl"
    SetEnv MP_ACCESS_TOKEN "..."
    SetEnv NEXT_PUBLIC_BASE_URL "https://vaulthost.cl"
    SetEnv NEXT_PUBLIC_MP_PUBLIC_KEY "..."
</IfModule>
```

4. Reconstruir y recargar:
```bash
/usr/local/cpanel/scripts/rebuildhttpdconf && systemctl reload httpd
```

---

### 3.7 — Configurar webhook MercadoPago 🛠️

En mercadopago.cl → Tu negocio → Integraciones → VaultHost → Webhooks:

| Campo | Valor |
|---|---|
| Modo | Productivo |
| URL de producción | `https://vaulthost.cl/api/webhook` |
| Eventos | ✅ Pagos · ✅ Órdenes comerciales |

---

## 4. Validación final ✅

| Check | Resultado |
|---|---|
| `https://vaulthost.cl` carga | ✅ |
| Todas las rutas (`/planes`, `/contacto`, etc.) | ✅ |
| Tema claro/oscuro persiste | ✅ |
| HTTPS con candado verde ("La conexión es segura") | ✅ AutoSSL activo |
| WhatsApp real en todos los archivos | ✅ `56971456955` |
| Webhook MercadoPago configurado | ✅ |
| Prueba de pago e2e | ⏳ Ticket 4 |

---

## 5. Modelo de aprendizaje — ¿Qué pasó y por qué fue difícil?

Esta sección existe para que entiendas la lógica detrás del problema, no solo los comandos.

---

### 5.1 — La analogía del restaurante

Imagina que Apache es el **maître** de un restaurante. Cuando llega un cliente (request), el maître decide qué hacer:

- Si ve una mesa con platos listos (archivos estáticos en `public_html`) → los sirve directamente.
- Si ve una orden especial que necesita cocina (app Node) → la pasa al cocinero (Passenger).

**El problema que tuvimos:** el maître (Apache) siempre veía los platos listos (`public_html`) antes de ver la orden especial. Nunca llegaba a llamar al cocinero (Passenger).

**La solución:** le quitamos los platos listos de la mesa (symlink `public_html` → `standalone`) y le pusimos un letrero claro: "Aquí hay una app Node, el startup file es `app.js`" (`PassengerAppType node` + `PassengerStartupFile`). Ahora el maître sabe que debe llamar al cocinero.

---

### 5.2 — El diagrama de flujo del request

```
ANTES (roto):
Usuario → Apache → busca public_html → encuentra archivos estáticos → sirve "Index of /"
                                                      ↑
                                              Passenger nunca activa

DESPUÉS (funcionando):
Usuario → Apache → lee .conf → ve PassengerAppType node → llama a Passenger
                                                                    ↓
                                                         Passenger ejecuta app.js
                                                                    ↓
                                                         app.js llama a server.js
                                                                    ↓
                                                         Next.js responde ✅
```

---

### 5.3 — Por qué el .htaccess no funciona para Passenger

El `.htaccess` es como una nota que pega el arrendatario en su puerta. Funciona para muchas cosas (redireccionamientos, contraseñas), pero **no para directivas de Passenger** — esas tienen que ir en el contrato principal (el virtualhost de Apache), que solo puede modificar el dueño del edificio (root/WHM).

Error exacto que nos confirmó esto:
```
PassengerEnabled cannot occur within htaccess files
```

**Lección:** Passenger se configura en el virtualhost, no en `.htaccess`.

---

### 5.4 — El "Node fantasma" y por qué usamos app.js

El `server.js` de Next.js standalone está diseñado para correr **solo** — escucha en un puerto (`3000` por defecto). Pero Passenger en modo Apache no usa puertos, usa **sockets Unix** (canales internos que no se exponen a internet).

La analogía: es como si Next.js dijera "me conecto por teléfono al número 3000" pero Passenger solo habla por **walkie-talkie** (socket). Nunca se podían comunicar.

El `app.js` que creamos es el **traductor**: le dice a Next.js "usa el walkie-talkie que te pasa Passenger (`process.env.PORT`)".

---

### 5.5 — Lo que aprendiste sobre el stack

```
Nivel 1 — Qué ya sabías antes:
  ✓ Git clone, npm ci, npm run build
  ✓ Variables de entorno en .env

Nivel 2 — Lo que aprendiste hoy:
  ✓ Cómo funciona Apache + Passenger + Node juntos
  ✓ Que las directivas Passenger van en virtualhost, no en .htaccess
  ✓ Que Next.js standalone necesita PassengerAppType node explícito
  ✓ Que el DocumentRoot de cPanel no se puede sobreescribir fácilmente
  ✓ Que la documentación oficial > suposiciones > adivinar

Nivel 3 — Lo que debes reforzar:
  ○ Leer logs de Apache antes de intentar soluciones (tail -20 /etc/apache2/logs/error_log)
  ○ Distinguir el error HTTP de la causa raíz (403 ≠ permisos, puede ser Passenger)
  ○ Entender qué hace rebuildhttpdconf y cuándo usarlo
```

---

### 5.6 — Hábito a construir: logs primero

El error más costoso de esta sesión fue intentar soluciones sin leer el log. Cuando lo leímos, el diagnóstico fue inmediato:

```
AH01276: Cannot serve directory .../standalone/: No matching DirectoryIndex found
```

Esa línea nos dijo exactamente que Apache estaba sirviendo el directorio como estático — Passenger no activaba.

**Regla de empresa:** ante cualquier error HTTP → primero `tail -50 /etc/apache2/logs/error_log`, después pensar en soluciones.

---

## 6. Problemas encontrados y resolución

| # | Problema | Causa raíz | Solución |
|---|---|---|---|
| P1 | Shell deshabilitado en `vaulthostweb` | cPanel deshabilita shell por defecto | `whmapi1 modifyacct shell=/bin/bash` |
| P2 | Git pedía password | GitHub eliminó auth por password en 2021 | Deploy key SSH |
| P3 | `Index of /` — Passenger no activaba | Apache servía `public_html` estático | Symlink + `.conf` con directivas Passenger |
| P4 | `500` — `.htaccess` con `PassengerEnabled` | Passenger no puede configurarse en `.htaccess` | Mover config al virtualhost |
| P5 | `403 Forbidden` | Permisos + Passenger sin `AppType node` | `chmod 755` + agregar `PassengerAppType node` |
| P6 | `Index of /` con archivos correctos | `PassengerStartupFile` y `AppType` faltaban | Configuración completa + `app.js` wrapper |
| P7 | Next.js no conecta con Passenger | `server.js` usa puerto, Passenger usa socket Unix | `app.js` wrapper que exporta correctamente |

---

## 7. Correcciones al runbook (v1.2)

- **§6 Paso 5 (Application Manager):** agregar que después del Deploy hay que crear manualmente el `.conf` en `/etc/apache2/conf.d/userdata/` con `PassengerAppType node` y `PassengerStartupFile app.js`.
- **§6 nuevo paso:** documentar la creación del `app.js` wrapper y el symlink `public_html` → `standalone`.
- **§6 nuevo paso:** documentar `rebuildhttpdconf` como paso obligatorio después de modificar los `.conf`.
- **§7 Hardening:** agregar tarea de restringir shell de `vaulthostweb` post-deploy.

---

## 8. Decisiones tomadas

| # | Decisión | Por qué |
|---|---|---|
| **D4** | Symlink `public_html` → `standalone` | Única forma de que Apache/Passenger encuentren la app sin modificar el `DocumentRoot` del virtualhost generado por cPanel |
| **D5** | Crear `app.js` wrapper | Next.js `server.js` usa puerto; Passenger necesita que la app sea interceptable por socket Unix |
| **D6** | `PassengerAppType node` explícito en `.conf` | Sin esto, Passenger no sabe que debe manejar la app como Node.js |

---

## 9. Lecciones aprendidas

- **Logs primero, soluciones después.** El error exacto estaba en el log desde el principio. Leerlo antes habría ahorrado varios intentos.
- **La documentación oficial es la fuente de verdad.** Los ejemplos de internet asumen contextos distintos (CloudLinux, Nginx, PHP). La doc oficial de Passenger dejó claro qué directivas se necesitaban.
- **cPanel regenera el httpd.conf.** Modificaciones manuales al `httpd.conf` se pierden. El lugar correcto es `/etc/apache2/conf.d/userdata/` + `rebuildhttpdconf`.
- **`PassengerAppType node` no es opcional.** Sin él, Passenger no intercepta la app aunque todo lo demás esté correcto.
- **Next.js standalone ≠ app Express estándar.** El `server.js` que genera no exporta el app object — necesita un wrapper para integrarse con Passenger correctamente.

---

## 10. English practice — glosario de la fase

| English | Español | Uso en esta fase |
|---|---|---|
| Reverse proxy | Proxy inverso | Apache reenvía requests a Passenger |
| Unix socket | Socket Unix | Canal interno entre Apache y Node, no expuesto a internet |
| Startup file | Archivo de inicio | El `app.js` que Passenger ejecuta para arrancar la app |
| Document root | Raíz del documento | El directorio que Apache sirve por defecto |
| Symlink | Enlace simbólico | `public_html` apuntando al `standalone` |
| App type | Tipo de aplicación | `node` — le dice a Passenger qué runtime usar |
| Virtual host | Virtualhost | Configuración de Apache para un dominio específico |
| Rebuild | Reconstruir | `rebuildhttpdconf` regenera el `httpd.conf` desde las configs de cPanel |

> *Practice:* "Passenger needs an explicit app type declaration to know it's a Node.js app. Without it, Apache serves the directory as static files instead of forwarding requests to the Node process through the Unix socket."
> *Práctica:* "Passenger necesita una declaración explícita del tipo de app para saber que es Node.js. Sin ella, Apache sirve el directorio como archivos estáticos en vez de reenviar los requests al proceso Node a través del socket Unix."

---

## 11. Estado final — Definition of Done del Ticket 3

- [x] Shell habilitado temporalmente para deploy (pendiente restringir post-Sprint 1)
- [x] Deploy key SSH configurada en GitHub
- [x] Repo clonado en `/home/vaulthostweb/vaulthost-web/`
- [x] WhatsApp placeholder reemplazado por número real (`56971456955`) en todos los archivos
- [x] `output: 'standalone'` en `next.config.ts`
- [x] `.env.production` creado con credenciales de producción (en `.gitignore`)
- [x] Build exitoso (`npm ci && npm run build`)
- [x] Estáticos copiados al standalone (`public/` y `.next/static/`)
- [x] App registrada en Application Manager (Enabled)
- [x] Symlink `public_html` → `.next/standalone`
- [x] `app.js` wrapper creado
- [x] `.conf` con `PassengerAppType node` + `PassengerStartupFile app.js`
- [x] `rebuildhttpdconf` + `systemctl reload httpd`
- [x] Sitio cargando en `https://vaulthost.cl` ✅
- [x] HTTPS con candado verde (AutoSSL) ✅
- [x] Webhook MercadoPago configurado (Pagos + Órdenes comerciales) ✅
- [ ] ⚠️ **Pendiente Ticket 4:** prueba de pago e2e con tarjetas de prueba MP

---

## 12. Próximos pasos

**Ticket 4 — Prueba e2e MercadoPago:**
- Obtener tarjetas de prueba oficiales de MP desde developers.mercadopago.com
- Realizar pago de prueba en modo productivo
- Verificar que el webhook recibe la notificación
- Verificar que el voucher PDF se genera correctamente

**Sprint 2 — Hardening:**
- Restringir shell de `vaulthostweb` a `/bin/false`
- Cloudflare delante del dominio (CDN + anti-DDoS)
- ModSecurity (WAF) — reglas OWASP CRS
- fail2ban
- Headers HSTS + CSP básica
- Auditoría Lynis

---

*Fin de la bitácora — Sprint 1 / Ticket 3 · VaultHost.cl*
