# Bitácora AGC — 25 de julio de 2026

## Módulo en construcción
Identidad (esqueleto backend: NestJS + Prisma)

## Objetivo de la sesión
Levantar el esqueleto funcional del backend (PrismaModule, AppController/AppService, main.ts con seguridad y Swagger) y cerrar decisiones pendientes de arquitectura de autenticación para el módulo Identidad.

## Decisiones de arquitectura (actualización Documento Maestro → v2.6)

1. **RF-ESG-EN-04b confirmado sin cambios** como método M (obligatorio) del lanzamiento (rendimiento estático × distancia). RF-ESG-EN-04 (telemetría GPS en tiempo real) se mantiene como hito post-1-año, pendiente de validación explícita con Francis. No hubo reversión de la decisión de v2.5.

2. **RF-USR-08 (nuevo/revisado):** autenticación por RUN (no email) como identificador base, para todos los roles. Para todo usuario con **cuenta registrada** (incluye Vecino registrado; excluye Vecino en modo invitado):
   - El sistema genera automáticamente una contraseña robusta (20+ caracteres) que el usuario nunca memoriza ni transcribe.
   - Inmediatamente después, la app fuerza la activación de biometría local del dispositivo (huella/rostro, vía `local_auth` de Flutter).
   - Fallback si no hay hardware biométrico o falla: (a) PIN local para el mismo dispositivo, (b) OTP por SMS para dispositivo nuevo/perdido/robado, que dispara un nuevo enrolamiento.
   - Razón del alcance ampliado a Vecino: su cuenta tiene valor real para él (saldo AGC Coins, cupones activos/vencidos/utilizados) — el riesgo de robo de cuenta no es menor solo porque no sea de alto valor a nivel empresarial.

3. **RF-USR-09 (nuevo):** roles administrativos (Supervisor_Municipal, Administrador_Municipal, Superadmin_AGC) exigen además 2FA obligatorio (TOTP/código temporal) sobre el esquema anterior.

4. **RF-FLOTA-02 (aclarado):** el Chofer autoselecciona la patente al iniciar sesión, de una lista filtrada que excluye vehículos con mantención activa o documentación vencida (RF-FLOTA-10). No la asigna el Supervisor — evita cuello de botella y usa datos que el sistema ya tiene.

## Código implementado hoy

- `backend/src/prisma/prisma.service.ts`: cambiado de `export default PrismaService` a `export class PrismaService` (named export), para alinear con la convención de NestJS y permitir autoimport/refactor consistente en los ~15 módulos futuros.
- `backend/src/prisma/prisma.module.ts`: corregido orden de imports/decoradores (`@Global()` + `@Module()` deben ir pegados a la clase, sin imports en medio); import de `PrismaService` con llaves.
- `backend/src/app.controller.ts` y `app.service.ts`: boilerplate base con inyección de dependencias por constructor (patrón que se repetirá en cada módulo de negocio).
- `backend/tsconfig.json`: estaba vacío — causa raíz de varios errores en cascada (incluyendo `Cannot find name 'process'`). Recreado completo con `experimentalDecorators`, `emitDecoratorMetadata`, `types: ["node"]`, y `rootDir: "./src"` (ajuste propio de Oscar sobre la base sugerida).
- `backend/src/main.ts`: bootstrap con orden Helmet → ValidationPipe → Swagger → listen. Dependencias nuevas instaladas: `helmet`, `@nestjs/swagger` (v10).

## Errores resueltos (aprendizajes para próximos módulos)

- Decoradores de Nest (`@Global()`, `@Module()`, `@Injectable()`, `@Controller()`) deben ir inmediatamente pegados a la clase que decoran — nunca con imports u otras declaraciones en medio.
- Un import con llaves incorrecto (`{ Nombre }` vs `Nombre`) depende exclusivamente de si el archivo origen usa `export class X` (named) o `export default X` (default) — no es una regla universal, hay que revisar el archivo fuente.
- `helmet` es default export (`import helmet from 'helmet'`), no named.
- Los submódulos internos de paquetes (ej. `@nestjs/common/pipes/validation.pipe`) no son rutas de import estables — siempre importar desde la raíz del paquete (`@nestjs/common`).
- `process` es un global de Node.js, no se importa — su tipado depende de `@types/node` y de que `tsconfig.json` tenga contenido válido (`types: ["node"]`).
- Errores en cascada («Cannot find module») desaparecen recién después de reiniciar el TS Server de VSCode (`Ctrl+Shift+P` → `TypeScript: Restart TS Server`), no solo con guardar el archivo.

## Pendiente para la próxima sesión

- Escribir el modelo `usuarios` en `schema.prisma` reflejando RUN, contraseña autogenerada, flags de biometría/PIN/2FA por rol (Identidad).
- Validar con Francis las dos variantes de RF-ESG-EN-04/04b antes de fijar columnas finales de combustible (Flota).
- Subir el Documento Maestro v2.5.1 → v2.6 real (.docx) al conocimiento del proyecto, con los RF-USR-08/09 y aclaración de RF-FLOTA-02 incorporados.
