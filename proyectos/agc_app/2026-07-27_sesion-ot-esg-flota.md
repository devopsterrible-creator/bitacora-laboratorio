# Bitácora de laboratorio — AGC Conecta Chile
**Fecha:** 27-07-2026
**Sesión:** Schema Prisma completo — OT/Flujo, ESG (energético+operativo), Flota

---

## 1. Resumen ejecutivo

Se cerró el schema de Prisma de los tres módulos restantes del ciclo operativo central: **OT/Flujo**, **ESG** y **Flota**. El `schema.prisma` quedó validado (`prisma validate` ✅) con ~29 modelos en total, cubriendo Identidad, Catastro, OT, ESG y Flota. **Pendiente crítico para la próxima sesión: correr `npx prisma migrate dev` — el schema está validado pero la base de datos real todavía no tiene las tablas de estos 3 módulos.**

Se recibió y analizó el documento "Flujo OT Corregido" de Francis Poblete (8 correcciones sobre el diseño previo), y se detectaron y corrigieron 2 gaps propios antes de que llegaran a producción.

## 2. OT/Flujo

- **`Falla` y `OrdenTrabajo` son entidades separadas.** `OrdenTrabajo.fallaId` es nullable — mantenimiento preventivo/programado puede generar una OT sin ningún reporte previo.
- **`TipoFalla`** es un catálogo con `esDesperdicioElectrico: Boolean` (reemplaza una lista de IDs hardcodeada en el servicio). **Pendiente de seed:** marcar `true` específicamente en los IDs legacy 2, 3 y 5 (luminaria encendida de día, circuito encendido de día, luminaria intermitente) — Corrección 6 de Francis, verificada contra OT-24.
- **Máquina de estados real:** `INGRESADA → APROBADA → EN_PROCESO → CONFIRMACION_TECNICA → APROBACION_VECINAL → FINALIZADO`, con ramas `RECHAZADA` y `SIGUE_EN_FALLA`.
- **Aprobación Vecinal:** cualquier Vecino registrado puede confirmar o rechazar (no necesariamente quien reportó — cubre el caso de reportes por llamado telefónico sin cuenta detrás). Plazo configurable (propuesto 5 días por defecto) con cierre automático si nadie responde.
- **`HistorialEstadoOT`** es la bitácora inmutable de cada transición, con `rolSnapshot` (copia fija del rol al momento del cambio, no referencia viva) y campos de evidencia (foto + descripción) para el caso de rechazo vecinal.
- **Autorización (pendiente de implementar en guards, no en schema):** el Maestro y el Chofer reciben ambos la notificación de la OT asignada, pero solo el Maestro puede transicionar a `CONFIRMACION_TECNICA`.

## 3. ESG — dos gaps propios detectados al revisar el documento de Francis

| Gap | Corrección |
|---|---|
| Corrección 7 de Francis ("ambos cálculos" deben tener snapshot de parámetros) solo se había aplicado a `AhorroEnergetico` | Se agregó a `AhorroOperativo`: `combustible`, `factorCombustibleUsado`, `precioCombustibleUsado`, `rendimientoUsado` |
| `OrdenTrabajo` no tenía ubicación propia cuando no hay `Falla` (preventivo) | Se agregó `luminariaId`/`luminaria` directo en `OrdenTrabajo` |

- **Regla de datos faltantes, distinta por módulo:** en `AhorroEnergetico`, falta un parámetro → excepción, se detiene el cálculo (RF-ESG-EN-07). En `AhorroOperativo`, falta un dato → se sigue con lo disponible y se deja una nota en `observaciones` (Corrección 8 de Francis, caso de referencia: "OT 39" del catastro legacy de Colina/Sinec — dato histórico a migrar, no un hueco del sistema nuevo).
- **`es_gasto`** en `AhorroOperativoDetalle` sigue siendo la frontera de negocio entre lo que ve el municipio y el costo interno del contratista.
- Cuando `fallaId` es null (preventivo), no se genera línea de hora-hombre evitada/detección — solo costos reales de cuadrilla, todos `es_gasto = true`.

## 4. Flota

- **Regla dura confirmada:** la cuadrilla siempre sale con vehículo — `SalidaTerreno.vehiculoId` pasó de nullable a obligatorio, con relación real a `Vehiculo`. Se eliminó el campo `patente` suelto (redundante, ahora vive en `Vehiculo.patente`).
- Modelos nuevos: `Vehiculo`, `ChecklistSalidaVehiculo` (RF-FLOTA-04, 1:1 con `SalidaTerreno`, bloquea acceso a OTs hasta `aprobado = true`), `EventoCombustible` (RF-FLOTA-08), `VisitaServicioVehiculo` (RF-FLOTA-09), `DocumentoVehiculo` (RF-FLOTA-10).
- `OrdenTrabajo.ordenEnRuta` agregado para que el Chofer reordene sus OT del día (RF-FLOTA-05).
- Rutas (Haversine, API externa de ruteo) y Android Auto (Fase 2) quedan fuera del schema a propósito — son lógica de servicio o alcance futuro.

## 5. Hábito detectado — Copilot autocompletando reglas que no existen

Dos veces en la sesión, Copilot Tab autocompletó comentarios con número de RF y regla de negocio (`RF-OT-08`, `RF-ESG-OP-06`, ambos sobre "visibilidad al vecino") que nunca fueron decisiones reales — se detectaron y eliminaron. Vale la pena revisar con más atención los comentarios que Copilot genera junto a los campos, especialmente cuando "suenan" a regla oficial.

## 6. Diseño aprobado para más adelante (no bloquea nada hoy)

Para el futuro módulo de Reportes/API pública: el Vecino vería un seguimiento simplificado tipo "estado de un paquete" (`Reportada → En proceso → Confirmada → Esperando tu aprobación → Cerrada`), no el `HistorialEstadoOT` interno completo.

## 7. Pendiente para la próxima sesión — en orden

1. **`npx prisma migrate dev --name init_ot_esg_flota`** — antes que cualquier otra cosa.
2. Seed de `TipoFalla` con los IDs reales 2/3/5 marcados `esDesperdicioElectrico = true`.
3. Actualizar Documento Maestro a v2.9 con las correcciones de Francis (nota de IDs en RF-ESG-EN-01, nueva RF de snapshot en operativo, nueva RF de "observación explícita, no cero oculto").
4. Recién después: empezar los **servicios de dominio en NestJS** — la máquina de estados de OT en código real. Hasta ahora todo lo construido es `schema.prisma`, sin una sola línea de servicio.
