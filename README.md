# Bitácora de Laboratorio Informático

Biblioteca personal de documentos instructivos sobre cómo trabajar de forma
**profesional** en proyectos de desarrollo y operaciones: Node, Next.js,
despliegue web + API, hardening de servidores, etc.

Mantenida con enfoque **docs as code**: todo en Markdown, versionado en Git.

## Estructura

- `_plantillas/` — plantillas reutilizables.
- `adr/` — Architecture Decision Records (decisiones de arquitectura).
- `proyectos/` — bitácora cronológica por proyecto (el registro crudo de cada fase).
- `guias/` — guías destiladas y reutilizables (material de estudio / enseñanza).
- `glosario.md` — términos técnicos ES/EN.

## Las dos capas

1. **proyectos/** = qué pasó realmente, fase por fase, con sus *porqués*.
2. **guias/** = el conocimiento destilado de esas bitácoras en instructivos genéricos.

La bitácora alimenta a las guías.

## Índice

### Proyectos
- [VaultHost.cl](proyectos/vaulthost/) — despliegue de sitio comercial Next.js + API.

### Guías
- _(en construcción)_

### ADR
- [ADR-001 — Opción A: cPanel Application Manager + Passenger](adr/ADR-001-opcion-a-passenger.md)

---
_Seguridad: este repo NUNCA contiene secretos. Ver `.gitignore`. Redactar IPs y credenciales reales con placeholders._
