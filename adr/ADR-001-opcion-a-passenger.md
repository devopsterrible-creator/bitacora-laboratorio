# ADR-001 — Despliegue con cPanel Application Manager + Passenger (Opción A)

- **Estado:** Aceptada
- **Fecha:** 24-05-2026
- **Proyecto:** VaultHost.cl

## Contexto
El sitio VaultHost.cl es una app Next.js (frontend + API routes en un solo
proceso Node). Debe correr en un VPS propio (AlmaLinux 9.7) que usa cPanel/WHM
con Apache en los puertos 80/443.

## Decisión
Usar la **Opción A**: cPanel Application Manager + Phusion Passenger (ea-nodejs).
cPanel administra el arranque, reinicio y proxy del proceso Node; Apache hace de
reverse proxy hacia un socket interno gestionado por Passenger.

## Alternativas descartadas
- **Reseller Nuthost:** no soporta Node.js de forma confiable.
- **next export estático:** rompe las API routes (Mercado Pago).
- **Nginx independiente:** chocaría con Apache en 80/443.
- **PM2 + /opt (Opción B manual):** más control, pero más mantenimiento del necesario.

## Consecuencias
- (+) Gestión automática del proceso → menos mantenimiento manual.
- (+) Encaja con el alcance acotado (web comercial, sin escalamiento ni soporte 24/7).
- (−) Depende de las versiones de ea-nodejs que ofrezca EasyApache (a verificar).
- (−) Menos control fino que un setup manual (aceptable para este alcance).
