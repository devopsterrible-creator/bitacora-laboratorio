# Bitácora de laboratorio — AGC Conecta Chile
**Fecha:** 27-07-2026
**Sesión:** Módulo Identidad + Catastro (schema, migración, seed) · Documento Maestro v2.6 → v2.8

---

## 1. Resumen ejecutivo

Se cerró de punta a punta el schema de Prisma de los módulos **Identidad** y **Catastro**: modelado, validado, migrado a PostgreSQL+PostGIS real, y sembrado (seed) con los 8 roles y sus permisos. El Documento Maestro de Requisitos subió tres versiones (v2.6 → v2.7 → v2.8) incorporando decisiones de arquitectura y el documento de trazabilidad del motor de ahorro entregado por Francis Poblete.

## 2. Identidad — decisiones cerradas

- **Autenticación por RUN** (no email), contraseña autogenerada (20+ caracteres), biometría local obligatoria (`local_auth` Flutter) para todo usuario registrado, con recuperación en 2 niveles (PIN local / OTP SMS). 2FA obligatorio para roles administrativos.
- **RBAC dinámico** (`Rol`, `Permiso`, `RolPermiso` como tablas reales, no enum fijo) — corrige una inconsistencia propia: se había propuesto un enum `Rol` que contradecía la sección 6 del Documento Maestro, que ya especificaba tablas `usuarios, roles, permisos`.
- **Seguridad de asignación de rol** (RF-USR-10/11): el registro público de Vecino nunca acepta `rol` desde el cliente — el servicio de dominio lo hardcodea buscando la fila `nombre = "vecino"`. Creación de roles operativos reservada a Administrador_Municipal (su propio tenant) y Superadmin_AGC. Un Administrador_Municipal no puede crear otro — evita escalada de privilegio horizontal. Alta de Administrador_Municipal requiere solicitud formal documentada.
- **RF-OT-04 deprecado**: ningún usuario no registrado puede reportar una falla — se elimina el modo invitado/anónimo del Vecino.
- Convención de naming cerrada para todo el proyecto: campos Prisma en camelCase, mapeados a columnas Postgres en snake_case vía `@map`; sin `@map` en valores de enum (evita bug conocido de Prisma 7.x, prisma/prisma#28843).

## 3. Catastro — modelo real vs. legacy

Se auditó la tabla real de Francis (no estaba documentada) y se corrigieron errores de tipo del sistema legacy antes de llevarlos al schema nuevo:

| Error legacy | Corrección |
|---|---|
| `id_catastro SERIAL` (overflow 32 bits, mismo bug que `usuarios`) | `BigInt` |
| `potencia VARCHAR` (mezclaba número + unidad) | `potencia` numérica por foco, en `LuminariaLampara` |
| `tipo_lampara_1/2` (columnas repetidas, máx. 2) | Tabla hija `LuminariaLampara` (N tipos) |
| `imagen BYTEA` (foto binaria en la BD) | `imagenUrl String` (archivo en disco/bucket) |
| `empalme_equipo_control VARCHAR` decía "booleano" pero no lo era | Entidad propia `EquipoControl`, con `tieneEmpalme Boolean` + `empalmeId` FK |
| `estado` de catastro con valores de ciclo de vida de falla | Separación `Luminaria.estado` vs `Falla.estado` |

Nuevas entidades: `EquipoControl` (empalme, ícono propio en el mapa, agrupa luminarias), `LuminariaLampara` (N focos por poste). Enums cerrados: `TipoPropiedad`, `TarifaElectrica`, `TipoAlimentacion`, `TipoConductor`.

## 4. Infraestructura

- **PostGIS**: la imagen `postgres:16-alpine` no traía la extensión — se cambió a `postgis/postgis:16-3.5-alpine` en `docker-compose.yml`. Se declaró `extensions = [postgis, postgis_topology, postgis_tiger_geocoder, fuzzystrmatch]` en el datasource (`previewFeatures = ["postgresqlExtensions"]`).
- **Migración aplicada**: `20260727200321_init_identidad_catastro` — 11 tablas creadas (`municipios_clientes`, `usuarios`, `roles`, `permisos`, `roles_permisos`, `equipos_control`, `luminarias`, `luminarias_lamparas`, `fallas`), con índices `Gist` espaciales.
- **Seed**: `prisma/seed.ts` — 8 roles + 8 permisos iniciales + asignaciones, todo con `upsert` (idempotente). `Superadmin_AGC` queda deliberadamente sin filas de permiso: se trata como bypass total de RBAC en el guard, no como acumulador de permisos.
- Fix de `tsconfig.json`: se agregó `"prisma"` al `exclude`, porque `rootDir: "./src"` entraba en conflicto con `prisma/seed.ts` (vive fuera de `src/`).
- `.env`: symlink resuelto entre `backend/.env` y `backend/prisma/.env` para que CLI y extensión de VS Code lean el mismo archivo.

## 5. Documento Maestro — versiones de hoy

- **v2.6**: RUN + biometría + 2FA, gap del rol Chofer corregido en RF-USR-02.
- **v2.7**: RF-OT-04 deprecado, RF-USR-10/11 nuevos, sección 3.2 Catastro reescrita completa (RF-CAT-01 a 07) con el modelo validado con Francis.
- **v2.8**: incorpora el documento "Trazabilidad Motor de Ahorro ESG" de Francis — fórmulas verificadas contra caso real OT-24, tres defectos conocidos documentados para NO replicar (horas desde fechas de OT, CO₂ evitado no guardado, estado de ahorro que no avanza), columna `es_gasto` como frontera de negocio municipio/contratista, y el diseño completo de Cuadrilla Habitual (plantilla) vs. Salida a Terreno (realidad, a la que se liga toda OT).

## 6. Pendiente para la próxima sesión

- Subir los `.docx` v2.6/v2.7/v2.8 al conocimiento del proyecto (reemplazando versiones anteriores).
- Modelar en Prisma el módulo **OT/Flujo**, ya informado por el documento de Francis: máquina de estados, `Falla` con `codigoCierre` y regla de material obligatorio, y las entidades `CuadrillaHabitual` / `SalidaTerreno`.
- Validar con Francis las dos variantes de RF-ESG-EN-04/04b antes de fijar columnas finales de ese cálculo (pendiente de sesiones anteriores, sigue abierto).
