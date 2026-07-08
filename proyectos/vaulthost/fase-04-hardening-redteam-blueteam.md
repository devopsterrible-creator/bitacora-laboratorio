# Bitácora — Sprint 2: Hardening (Red Team / Blue Team)

> **Documento de bitácora de fase — consolidado vivo.** Registra, ticket por ticket, el trabajo del Sprint 2 (Hardening) del proyecto VaultHost.cl. Se actualiza a medida que se cierra cada ticket SEC-XX del sprint.
> Complementa al runbook oficial (*VaultHost — Arquitectura y Runbook de Despliegue*).

| Campo | Valor |
|---|---|
| **Proyecto** | VaultHost VPS — sitio comercial vaulthost.cl |
| **Sprint** | Sprint 2 (Hardening) |
| **Metodología** | Red Team (hallazgos) → Blue Team (mitigación), tickets pareados |
| **Responsable** | Oscar (owner, dev y administrador) |
| **Acompañamiento** | Claude (ingeniero de cabecera) |
| **Estado del sprint** | En curso — 2 de 13 tickets cerrados |
| **Reglas de enganche** | Solo reconocimiento y escaneo de vulnerabilidades (pasivo/no-destructivo). No se generan ni almacenan exploits armados, brute-forcers ni payloads de exfiltración. Entregable: findings + remediación, no armamento. Evidencia guardada en `~/pentest-vaulthost/`. |

---

## Control de versiones del documento

| Versión | Fecha | Cambio | Autor |
|---|---|---|---|
| 1.0 | 07-07-2026 | Bitácora inicial: cierre de SEC-01 (recon) y SEC-02 (cerrar 3306) | Oscar + Claude |

---

## 1. Objetivo del sprint

Descubrir la superficie de ataque real del VPS y del sitio vaulthost.cl mediante reconocimiento autorizado (Red Team), construir un modelo de amenazas, e implementar las mitigaciones priorizadas (Blue Team), confirmando su efectividad con re-escaneo al cierre del sprint.

## 2. Checklist del sprint (Trello — Sprint 2 Hardening)

```
SEC-01 [RED] Nmap recon: puertos, servicios, banner grabbing            → ✅ CERRADO
SEC-02 [BLUE] Cerrar puerto 3306 (MySQL) a nivel firewall                → ✅ CERRADO
SEC-03 [BLUE] Cloudflare: proxy activo + ocultar IP origen  ⭐ máxima     → pendiente
SEC-04 [RED] TLS/SSL check (testssl.sh)                                  → pendiente
SEC-05 [BLUE] Headers HSTS + CSP                                        → pendiente
SEC-06 [RED] OWASP ZAP passive scan sobre vaulthost.cl                  → pendiente
SEC-07 [BLUE] ModSecurity (WAF) + rate-limit en /api/webhook            → pendiente
SEC-08 [RED] Nikto liviano + revisión de banners de versión             → pendiente
SEC-09 [BLUE] fail2ban (SSH + login cPanel)                             → pendiente
SEC-10 [BLUE] Verificar shell de vaulthostweb restringida               → pendiente
SEC-11 Consolidación: esta bitácora                                     → en curso
SEC-12 [BLUE] Deshabilitar SSH password authentication (solo llave)     → pendiente (agregado post SEC-01)
SEC-13 [BLUE] Restringir puertos WHM/cPanel/Webmail a VPN-only          → pendiente (agregado post SEC-01)
```

---

## TICKET SEC-01 — Reconocimiento (Red Team)

**Tipo de trabajo:** 🔍 Solo-lectura (escaneo pasivo/no-destructivo).
**Estado:** ✅ Cerrado

### Objetivo

Confirmar el estado real de la superficie de ataque del VPS desde una perspectiva externa (como lo vería un atacante desde internet), antes de decidir qué mitigar.

### Setup del entorno de pentest

- Kali Linux en VirtualBox, modo NAT — para tener una perspectiva verdaderamente externa (no desde dentro de la red del VPS).
- Confirmación de IP pública propia y recon pasivo de DNS/WHOIS.
- IP objetivo confirmada (pre-Cloudflare): **158.69.48.4**, registrada a OVH Hosting Inc. (Canadá).

### Hallazgos — puertos y servicios detectados

| Puerto | Servicio | Versión |
|---|---|---|
| 22 | SSH | OpenSSH 9.9 |
| 25 | SMTP | Exim 4.99.4 |
| 53 | DNS | PowerDNS 4.9.16 |
| 80 / 443 | HTTP/HTTPS | Apache |
| 587 / 993 / 995 | Mail stack (submission/IMAPS/POP3S) | — |
| 2083 / 2087 / 2096 | Paneles cPanel/WHM/Webmail | — |
| 3306 | MariaDB | 10.11.18 |
| 1723 | PPTP (sospechoso) | — |

### Hallazgos — puntos críticos identificados

1. **MariaDB (3306) bindeado a `0.0.0.0`** → expuesto directamente a internet. Hallazgo crítico, origen del ticket SEC-02.
2. **Antipatrón repetido de firewall:** las rich-rules de allowlist (6 IPs de Google Cloud AppSheet + subred WireGuard `10.8.0.0/24`) estaban siendo anuladas por aperturas blanket de puerto en la zona pública — el mismo patrón afecta a **3306, 2087 y 22**.
3. **`firewalld` usa backend `nftables`:** `iptables -L` da una lectura engañosa en este servidor; el estado real solo se confirma con `firewall-cmd --list-all`.
4. **SSH con `PasswordAuthentication yes`** → riesgo de fuerza bruta. `PermitRootLogin without-password` sí está correcto (root solo por llave).
5. **Docker con contenedor PostgreSQL** (proyecto Fluxyard) — confirmado seguro, bindeado solo a `127.0.0.1:5432`, sin exposición externa.
6. **PowerDNS confirmado no-recursivo** (safe).
7. **Exim 4.99.4 confirmado parchado** contra CVEs críticos recientes conocidos a la fecha.
8. **Puerto 1723 (PPTP):** marcado como posible falso positivo, pendiente de confirmación en un re-escaneo futuro.

### Problema encontrado durante el ticket

- Un comando `dig` inicial quedó mal formado (faltaba el operador `@` para apuntar al servidor DNS a consultar), lo que habría producido un test de recursividad DNS inválido. Se detectó y corrigió antes de sacar conclusiones sobre PowerDNS.

### Threat model consolidado (priorización Red→Blue)

| Ticket | Hallazgo (Red) | Mitigación (Blue) | Prioridad |
|---|---|---|---|
| SEC-02 | MySQL 3306 expuesto a `0.0.0.0/0` | Cerrar puerto a nivel firewall, preservando allowlist | Alta |
| SEC-03 | IP real del VPS visible desde internet | Cloudflare proxy + ocultar origen | **Máxima** |
| SEC-05 | Headers de seguridad faltantes | HSTS + CSP | Media |
| SEC-07 | Sin WAF activo | ModSecurity + rate-limit en `/api/webhook` | Alta |
| SEC-09 | Superficie de fuerza bruta (SSH + cPanel) | fail2ban | Media |
| SEC-10 | Shell de `vaulthostweb` habilitada desde el deploy (Ticket 3) | Restringir a `noshell` | Media |
| SEC-12 | SSH con password auth habilitado | Deshabilitar, forzar solo llave | Alta |
| SEC-13 | Puertos de panel (2083/2087/2096) expuestos a `0.0.0.0/0` | Restringir a VPN-only | Alta |

**Por qué Cloudflare (SEC-03) va primero entre las mitigaciones, aunque no sea el primer hallazgo cronológico:** una vez que Cloudflare oculta la IP real del VPS, todo el reconocimiento externo que cualquiera intente desde internet deja de ver el servidor real — solo ve Cloudflare. Es la mitigación con mayor efecto dominó: conviene aplicarla temprano y re-escanear después para confirmar que el resto de hallazgos ya no son visibles desde afuera.

### Estado final — Definition of Done SEC-01

- [x] Barrido completo de puertos ejecutado
- [x] Detección de servicios y versiones (banner grabbing) ejecutada
- [x] Threat model construido y priorizado
- [x] Antipatrón de firewall (rich-rules anuladas por reglas blanket) identificado y generalizado a 3 puertos (3306, 2087, 22)
- [x] Checklist de tickets Blue Team generado y cargado a Trello

---

## TICKET SEC-02 — Cerrar puerto 3306 (MariaDB) expuesto al mundo

**Tipo de trabajo:** 🛠️ Modificador (cambio real de firewall en producción).
**Estado:** ✅ Cerrado

### Objetivo

Cerrar la exposición pública del puerto 3306/tcp a nivel de firewall (`firewalld`, backend `nftables`), preservando el acceso legítimo de las IPs oficiales de AppSheet y de la VPN WireGuard interna (`10.8.0.0/24`).

**Problema raíz (heredado de SEC-01):** una regla de `firewalld` en blanco abría 3306 a `0.0.0.0/0`, anulando la allowlist existente de rich-rules. La fix debía ser quirúrgica: remover la apertura blanket sin tocar las rich-rules válidas ni hacer bind a `localhost` (rompería la integración con AGC).

### Punto de partida

- `ports:` incluía `3306/tcp` abierto a cualquier origen.
- Rich-rules existentes: 6 reglas individuales sueltas permitiendo IPs puntuales de AppSheet + 3 reglas de WireGuard (puertos 22, 2087, 3306).
- Fuente de la allowlist: archivo `AppSheet IP Address (all regions).xlsx` aportado por el cliente, con 73 IPs oficiales de Google/AppSheet.

### Auditoría de la fuente de datos (previa a tocar el servidor)

Antes de construir la allowlist se auditó el Excel de origen y se encontraron dos problemas de integridad de datos:

1. **28 IPs corrompidas:** Excel las autoconvirtió a número, perdiendo los puntos (ej. `34116117132` en vez de `34.116.117.132`). Se reconstruyeron aplicando la restricción de que un octeto IPv4 válido es 0–255 sin ceros a la izquierda; cada una tuvo una única reconstrucción matemáticamente posible, sin ambigüedad.
2. **1 fila duplicada que ocultaba una IP faltante:** `34.87.159.166` aparecía dos veces. Al comparar contra la documentación oficial de Google (`support.google.com/appsheet/answer/10104492`), se confirmó que la lista real trae `34.87.159.166` **y** `34.87.233.115` como IPs distintas y consecutivas. El error fue una copia accidental de fila, no una fila de más.

**Verificación cruzada:** se validó la lista final (73 IPs) contra la fuente oficial de Google usando una segunda IA (Perplexity) como control cruzado independiente, confirmando coincidencia exacta 1 a 1.

### Pasos ejecutados

**Pre-check (read-only):**
```bash
firewall-cmd --list-all
ss -tnp | grep :3306
firewall-cmd --get-ipsets
```

**Construcción de la allowlist vía ipset** (en vez de 73 rich-rules individuales, por mantenibilidad):
```bash
firewall-cmd --permanent --new-ipset=appsheet-ips --type=hash:ip
# carga de las 73 IPs verificadas
while read -r ip; do
  firewall-cmd --permanent --ipset=appsheet-ips --add-entry="$ip"
done < appsheet_ips_clean.txt
firewall-cmd --permanent --add-rich-rule="rule family='ipv4' source ipset='appsheet-ips' port port='3306' protocol='tcp' accept"
```

**Cierre de la exposición pública:**
```bash
firewall-cmd --permanent --remove-port=3306/tcp
firewall-cmd --reload
```

**Limpieza final:** remoción de las 6 rich-rules individuales antiguas, redundantes porque esas mismas IPs ya quedaron cubiertas dentro del ipset de 73.

### Problemas encontrados durante la ejecución

**Incidente 1 — orden de ejecución invertido:** por error de copiado, el bloque de cierre (`remove-port` + `reload`) se ejecutó **antes** que el bloque de construcción del ipset. Esto dejó temporalmente 3306 cerrado al público pero con el ipset aún vacío, cubierto solo por las 6 reglas individuales antiguas + WireGuard. Riesgo: integraciones de AppSheet desde IPs fuera de esas 6 podían fallar temporalmente.

**Incidente 2 — archivo faltante en el servidor:** el script se cortó en el paso de carga de IPs porque `appsheet_ips_clean.txt` no había sido subido al VPS. Se resolvió recreando el archivo directamente en el servidor vía heredoc (`cat > archivo << EOF ... EOF`) y retomando desde el paso donde se cortó.

Ambos incidentes se detectaron auditando el output de terminal antes de continuar, evitando dejar el servidor en un estado inconsistente sin darse cuenta.

### Verificación final (estado confirmado en el servidor)

```
rich rules:
        rule family="ipv4" source address="10.8.0.0/24" port port="3306" protocol="tcp" accept
        rule family="ipv4" source ipset="appsheet-ips" port port="3306" protocol="tcp" accept
        rule family="ipv4" source address="10.8.0.0/24" port port="2087" protocol="tcp" accept
        rule family="ipv4" source address="10.8.0.0/24" port port="22" protocol="tcp" accept
```
- `ports:` ya no incluye `3306/tcp`.
- `ipset appsheet-ips`: 73 entradas, verificadas 1 a 1 contra la lista oficial de Google.
- 0 rich-rules redundantes.

### Decisiones tomadas

| # | Decisión | Por qué |
|---|---|---|
| D7 | Usar un ipset de 73 entradas en vez de 73 rich-rules individuales | Mantenibilidad y legibilidad de la configuración |
| D8 | No hacer bind de MariaDB a `localhost` | Rompería la integración con AGC (AppSheet + PostgreSQL) |
| D9 | Remover las rich-rules individuales antiguas | Quedaban cubiertas por el ipset; evitar duplicidad confusa |

### Lecciones aprendidas

- Antes de correr un bloque de comandos con pasos irreversibles, revisar el **orden de ejecución** línea por línea, no solo el contenido.
- Verificar que todos los archivos necesarios (como listas de IPs) estén efectivamente presentes en el servidor **antes** de ejecutar un script que depende de ellos.
- Auditar la integridad de cualquier dataset externo (como un Excel con IPs) antes de usarlo para construir reglas de seguridad — Excel puede corromper datos silenciosamente al autoconvertir formatos.
- Cruzar una allowlist crítica contra la fuente oficial antes de darla por buena, incluso si la reconstrucción de datos parece matemáticamente sólida.

### Estado final — Definition of Done SEC-02

- [x] Apertura blanket de 3306/tcp a `0.0.0.0/0` removida
- [x] Allowlist AppSheet completa y verificada (73 IPs) implementada vía ipset
- [x] Rich-rule de WireGuard intacta
- [x] Rich-rules individuales redundantes limpiadas
- [x] Verificación final de `firewall-cmd --list-all` conforme a lo esperado

---

## 3. English practice — glosario acumulado del sprint

| English | Español | Uso en este sprint |
|---|---|---|
| Reconnaissance | Reconocimiento | Fase de descubrimiento pasivo (SEC-01) |
| Attack surface | Superficie de ataque | Todo lo expuesto a internet, mapeado en SEC-01 |
| Threat model | Modelo de amenazas | Tabla hallazgo → mitigación → prioridad |
| Rules of engagement | Reglas de enganche | Límites acordados antes de escanear |
| Mitigation | Mitigación | La respuesta Blue Team a cada hallazgo Red |
| Antipattern | Antipatrón | Rich-rules anuladas por reglas blanket, repetido en 3 puertos |
| Surgical fix | Fix quirúrgico | Cambio mínimo y preciso, sin tocar lo que funciona |
| Allowlist | Lista blanca | Las IPs de AppSheet + WireGuard permitidas en 3306 |
| Redundant rule | Regla redundante | Las 6 rich-rules individuales limpiadas al final de SEC-02 |

> *Practice:* "The threat model prioritizes surgical fixes over broad changes — closing exposure without breaking legitimate integrations."
> *Práctica:* "El modelo de amenazas prioriza fixes quirúrgicos por sobre cambios amplios — cerrar la exposición sin romper integraciones legítimas."

---

## 4. Próximos pasos

**Siguiente ticket:** SEC-03 — Cloudflare (proxy activo + ocultar IP de origen), marcado como prioridad máxima porque neutraliza el reconocimiento externo del resto del sprint.

**Pendiente de agregar a Trello (originados en SEC-01, no tenían tarjeta propia):**
- SEC-12 — Deshabilitar SSH password authentication (forzar solo llave)
- SEC-13 — Restringir puertos WHM/cPanel/Webmail (2083/2087/2096) a VPN-only

**Al cierre completo del sprint (Fase 4 — re-scan de confirmación):**
| Hallazgo original | Test de confirmación | ¿Cerrado? |
|---|---|---|
| IP expuesta | `dig +short vaulthost.cl` → debe dar IP de Cloudflare | ⬜ |
| MySQL 3306 | `nmap -p 3306 <ip>` → debe dar closed/filtered | ✅ (pendiente re-confirmar post-Cloudflare) |
| Headers | `curl -I https://vaulthost.cl` → HSTS + CSP presentes | ⬜ |

---

*Documento vivo — se actualiza al cierre de cada ticket del Sprint 2. Última actualización: SEC-02 · VaultHost.cl*
