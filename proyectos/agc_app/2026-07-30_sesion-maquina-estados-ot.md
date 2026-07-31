# Bitácora de laboratorio — AGC Conecta Chile

**Fecha:** 30-07-2026
**Repo código:** `~/dev/agc_app` (WSL) · **Repo bitácora:** `C:\ProyectosVSC\bitacora-laboratorio\proyectos\AGC_app\` (Git Bash)

---

## Resumen ejecutivo

Día largo y de alto rendimiento: se cerró la migración pendiente del schema (29 modelos aplicados a la BD real), se construyó y sembró el catálogo completo de `TipoFalla` (66 registros legacy, revisados uno por uno), y se destrabó una ambigüedad crítica heredada de Francis sobre la máquina de estados real de la OT — pasó de 6 a 7 pasos, con corrección del punto de disparo del cálculo ESG. Se escribió el primer código real del proyecto (`ot.service.ts`, método `reportarFalla()`). Documento Maestro avanzó de v2.8 a v2.11. Queda pendiente para mañana un bloque de diseño nuevo y grande: el mapa como mecanismo central de interacción (reporte, seguimiento, ruta).

---

## 1. Migración pendiente de ayer — resuelta

- Se corrió `npx prisma migrate dev --name init_ot_esg_flota` → aplicó los 29 modelos (Identidad, Catastro, OT/Flujo, ESG, Flota) que existían en schema.prisma pero no en la BD real.
- Verificado visualmente en Prisma Studio: `SalidaTerreno.choferId` y `SalidaTerreno.vehiculoId` quedaron obligatorios (no nullable), confirmando la regla de negocio de ayer (chofer nunca sale sin vehículo asignado).

## 2. Catálogo `TipoFalla` — análisis completo y seed

- Se revisaron las 66 filas reales de la tabla legacy `fallas` (phpMyAdmin), una por una.
- Se agregó el campo `categoria` (enum `CategoriaFalla`: REGULAR, INCIDENCIA, AMBIENTAL, URGENCIA) y el campo `activo` (Boolean) al modelo `TipoFalla`.
- **21 IDs excluidos** del alcance actual (trabajo técnico especializado — pintar, limpiar, corrosión, tapas, traslado de gancho, etc.) — se siembran con `activo=false`, nunca se borran, para preservar integridad referencial de OTs históricas que Francis pueda migrar después.
- **Categoría URGENCIA** creada para postes energizados (19, 44, 45) — validada con Francis con caso real de electrocución animal.
- **Categoría AMBIENTAL** creada para interferencia del entorno (hoy solo id 9 — follaje) — se mantiene aunque tenga un solo miembro, se espera que crezca.
- `esDesperdicioElectrico=true` confirmado en IDs 2, 3, 5, 65.
- Seed integrado en el único archivo `prisma/seed.ts` existente (no se creó un archivo seed separado — se descartó el borrador inicial `tipofalla.seed.ts` que no seguía el patrón real del proyecto).
- Migraciones aplicadas: `add_categoria_tipofalla`, `add_activo_tipofalla`.
- Verificado en Prisma Studio: 66 filas, categorías y flags correctos.

## 3. Máquina de estados de la OT — corrección mayor (6 → 7 pasos)

Se destrabó una ambigüedad real en la documentación de Francis (su documento escrito decía que el Maestro cierra la OT; Oscar recordaba una doble confirmación distinta). Tras conversación directa con Francis, quedó confirmado:

**Flujo real de 7 pasos:**
`Ingresada → Aprobada → En_Proceso → Confirmación_Técnica → Aprobación_Supervisor → Aprobación_Vecinal → Finalizado`

- **Confirmación_Técnica** (Maestro declara cierre) y **Aprobación_Supervisor** (Supervisor revisa y aprueba) son dos pasos distintos, ejecutados por dos actores distintos.
- **El cálculo ESG se fija en Aprobación_Supervisor**, no en Confirmación_Técnica (corrección importante respecto a lo documentado en v2.8/v2.9).
- **`Sigue_en_Falla` NO es un estado de reposo** — se eliminó del enum `EstadoOT`. Siempre desemboca en `Finalizado`:
  - Rechazo del Supervisor (antes del ESG) → revierte directo a `En_Proceso`, misma OT, sin estado propio.
  - Reapertura del Vecino (después del ESG) → la OT cierra igual como `Finalizado` con `OrigenCierreVecinal = REABIERTA_SIGUE_EN_FALLA`, y se genera una **OT completamente nueva**, vinculada por `otOrigenId` — nunca se reabre el ciclo original.
- Ningún cálculo ESG se anula ni se recalcula jamás — se marca `requirioReapertura=true` para que los reportes lo excluyan del "valor generado" sin perder el historial.
- Hora-hombre debe sumar **todos** los segmentos `En_Proceso` de una OT (no solo el último), si hubo rechazo y reintento.
- GPS-gating (RF-OT-11): botón "Iniciar" nunca bloquea por señal GPS al lanzamiento (evita bloquear al Maestro por mala señal); se registra `fueraDeRadio` como bandera de auditoría (radio configurable, default 40 m, rango 30–50 m). Bloqueo duro queda como mejora futura (Should), condicionado a evidencia real de precisión GPS en piloto.
- Autocierre de Aprobación_Vecinal: 72 horas (parámetro configurable `plazoAprobacionVecinalHoras`).
- Dos canales de notificación distintos: vista en vivo (todos los actores de una misma OT) vs. notificación dirigida por correo/push (solo el actor puntual relevante).
- Decisión de arquitectura para más adelante: `NotificationService` debe replicar el patrón de cola+reintentos que ya usaba el legacy (tabla `registro_cambio_estado_nuevo` + cron), pero con BullMQ/Redis en vez de un cron artesanal — para no bloquear una transición de estado si el envío de notificación falla.
- Separación de coordenadas lat/lng para exportación ArcGIS/AutoCAD (pedido de lujo de Francis): **no se replica el trigger legacy de duplicar columnas** — se resuelve con `ST_X()`/`ST_Y()` de PostGIS al momento de exportar el CSV, cuando lleguemos al módulo de Reportes. Cero columnas nuevas, cero riesgo de desincronización.

### Documento Maestro — versiones generadas hoy

| Versión | Contenido principal |
|---|---|
| v2.9 | Catálogo TipoFalla (`categoria`, `activo`), RF-OT-08/09/10, RF-ESG-OP-08/09 (snapshot de parámetros, observación explícita) |
| v2.10 | Máquina de 7 estados, RF-OT-11/12/13 (GPS-gating, autocierre 72h, vista en vivo vs. notificación), corrección del punto de disparo ESG |
| v2.11 | Corrección: `Sigue_en_Falla` eliminado del enum, mecánica de reapertura completa (RF-OT-14), regla de hora-hombre multi-segmento (RF-OT-15) |

Los tres `.docx` fueron entregados en el chat — **recuerda subirlos al proyecto para reemplazar la v2.8 anterior.**

## 4. Cambios de schema aplicados hoy (además del catálogo TipoFalla)

- `enum EstadoOT`: agregado `APROBACION_SUPERVISOR`, eliminado `SIGUE_EN_FALLA`.
- `enum OrigenCierreVecinal`: agregado `REABIERTA_SIGUE_EN_FALLA`.
- `HistorialEstadoOT`: agregado campo `fueraDeRadio Boolean?`.
- `OrdenTrabajo`: agregada auto-relación `otOrigenId` / `otReabiertaComo` para encadenar reaperturas.
- Migración final aplicada: `maquina_7_estados_reapertura`.
- **Pendiente de agregar** (mencionado, no confirmado si ya se migró): `requirioReapertura Boolean @default(false)` en `AhorroEnergetico` y `AhorroOperativo` — **verificar mañana si esto quedó aplicado**, no se vio confirmación explícita del `grep`/migración para este campo específico en el chat de hoy.

## 5. Primer código real del proyecto

Estructura creada en `src/ot/`:
```
src/ot/
├── dto/
│   └── reportar-falla.dto.ts   ✓ completo
├── ot.controller.ts             ✓ esqueleto mínimo (con placeholder inseguro pendiente de JWT)
├── ot.module.ts                 ✓ completo
└── ot.service.ts                ✓ método reportarFalla() completo
```

- Se instalaron paquetes faltantes: `class-validator`, `class-transformer`.
- `reportarFalla()` implementa el patrón de 3 pasos acordado (validar → transacción Prisma → notificar) para toda transición futura.
- **Roles que pueden reportar** (ampliado hoy): Vecino, Maestro Eléctrico, Chofer, Supervisor Municipal, Supervisor Mantenimiento — con regla de seguridad tipo RF-USR-10: `tipoFallaId` nunca se acepta tal cual del Vecino (se fuerza `null` en backend), sí es obligatorio y validado (`activo=true`) para los 4 roles técnicos.
- **Pendiente crítico de seguridad, marcado explícitamente en el código:** `ot.controller.ts` tiene `reportanteId` y `rolReportante` **hardcodeados** como placeholder — falta conectar el guard/decorador de autenticación real (JWT) del módulo Identidad. **Este es el primer punto a resolver mañana antes de seguir con el siguiente método de la máquina de estados.**

## 6. Pendiente para mañana — bloque de diseño nuevo (no iniciado)

Surgió al final del día, desde una conversación de Oscar con Francis: **el mapa no es una vista más — es el mecanismo central de interacción de toda la app**, no solo del catastro.

- El Vecino reporta seleccionando directamente el pin de la luminaria en el mapa (reconoce visualmente la falla, la selecciona, ve su historial/detalle, botón `+` abre el formulario ya con la luminaria identificada) — no busca por dirección en un formulario separado.
- Mapa único compartido: el mismo pin cambia de color/información en tiempo real según el estado de la OT, visible para todos los actores de esa OT, cada uno con sus permisos de rol.
- Mapa y tablas colapsables coexisten, no son alternativas — cada rol combina ambos según lo que necesita resolver en pantalla de teléfono:
  - Supervisor: tabla para priorizar/ordenar cola de OT entrantes (vista "buzón"), + mapa de catastro, + dashboard, + configuraciones.
  - Maestro: tabla para reordenar su ruta manualmente (o pedir ruta automática a Google Maps), + mapa para seguir esa ruta luminaria por luminaria, marcando cada una al completarla.
- RF-CAT-04 (paginación por bounding box del viewport) y RF-CAT-06 (Equipo de Control en mapa) ya cubrían parte del mecanismo técnico — lo nuevo es el mapa de OT como hub compartido en tiempo real, que no estaba documentado.
- Se decidió explícitamente dejar este diseño para una sesión propia mañana, con su propio diagrama, en vez de resolverlo apurado al final de una sesión ya muy densa.

---

## Comandos ejecutados hoy (referencia rápida)

```bash
npx prisma migrate dev --name init_ot_esg_flota
npx prisma migrate dev --name add_categoria_tipofalla
npx prisma migrate dev --name add_activo_tipofalla
npx prisma db seed
npx prisma migrate dev --name maquina_7_estados_reapertura
npm install class-validator class-transformer
```

## Primera tarea de mañana (sugerida)

1. Verificar si `requirioReapertura` quedó migrado en `AhorroEnergetico`/`AhorroOperativo` (punto suelto de hoy).
2. Conectar autenticación real (JWT) en `ot.controller.ts` — reemplazar el placeholder inseguro.
3. Seguir con el segundo método de la máquina de estados (`aprobar()`, Ingresada → Aprobada).
4. Arrancar el bloque de diseño del mapa como mecanismo central (nueva sesión, con diagrama propio).
